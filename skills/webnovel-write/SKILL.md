---
name: webnovel-write
description: 产出可发布章节，完整执行上下文→起草→审查→润色→提交→备份。
allowed-tools: Read Write Edit Grep Bash Agent
---

# 写章流程

## 目标

产出可发布章节到 `正文/第{NNNN}章-{title}.md`。默认 2000-2500 字，用户/大纲另有要求时从之。

## 模式

| 模式 | 流程 |
|------|------|
| 默认 | Step 1→2→3→4→5→6 |
| `--fast` | Step 1→2→3(轻量)→4→5→6 |
| `--minimal` | Step 1→2→4(仅排版)→5→6 |

## 硬规则

- 禁止并步、跳步、伪造审查
- 必须使用 `Agent` 工具调用指定 subagent；不得用主流程口头代替 subagent 输出
- blocking issue 未解决不进 Step 4/5
- 失败只补跑失败步骤，不回退
- 参考资料按步骤按需加载

## 优先级

用户要求 > 状态机硬门槛 > 项目约束（总纲/设定/记忆）> skill 流程 > reference 建议

## CSV 检索（Step 2 按需）

```bash
python -X utf8 "${SCRIPTS_DIR}/reference_search.py" --skill write --table {表名} --query "{关键词}" --genre {题材}
```

触发条件：新角色→命名规则，战斗→场景写法，多角色对话→写作技法，情感描写→写作技法，高频桥段→场景写法。

## 执行流程

### 准备：预检

```bash
export WORKSPACE_ROOT="${CLAUDE_PROJECT_DIR:-$PWD}"
export SCRIPTS_DIR="${CLAUDE_PLUGIN_ROOT:?}/scripts"
export SKILL_ROOT="${CLAUDE_PLUGIN_ROOT:?}/skills/webnovel-write"

python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${WORKSPACE_ROOT}" preflight
export PROJECT_ROOT="$(python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${WORKSPACE_ROOT}" where)"

python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" placeholder-scan --format text
```

### 准备：题材检测

```bash
if [ -f "${PROJECT_ROOT}/.webnovel/state.json" ]; then
    GENRE="$(python -X utf8 -c "import json; s=json.load(open('${PROJECT_ROOT}/.webnovel/state.json',encoding='utf-8')); print(s.get('project',{}).get('genre',''))")"
    PLATFORM="$(python -X utf8 -c "import json; s=json.load(open('${PROJECT_ROOT}/.webnovel/state.json',encoding='utf-8')); print(s.get('project',{}).get('platform',''))")"

    if [ "$GENRE" = "知乎短篇" ] || [ "$PLATFORM" = "知乎盐选" ]; then
        export NOVEL_TYPE="zhihu-short"
    else
        export NOVEL_TYPE="long-form"
    fi
elif [ -f "${PROJECT_ROOT}/config.json" ]; then
    PLATFORM="$(python -X utf8 -c "import json; print(json.load(open('${PROJECT_ROOT}/config.json',encoding='utf-8')).get('platform',''))")"
    if [ "$PLATFORM" = "知乎盐选" ]; then
        export NOVEL_TYPE="zhihu-short"
    else
        export NOVEL_TYPE="long-form"
    fi
else
    export NOVEL_TYPE="long-form"
fi

echo "Detected novel_type: ${NOVEL_TYPE}"
```

### 准备：刷新合同树

genre 从 `.webnovel/state.json` 的初始化配置快照读取，用于刷新合同树；写前主链真源仍是 `.story-system/` 合同。调用 story-system 前必须先从详细大纲解析真实本章目标，禁止传 `{章纲目标}`、`第N章章纲目标` 等占位 query。

```bash
GENRE="$(python -X utf8 -c "import json,sys; s=json.load(open('${PROJECT_ROOT}/.webnovel/state.json',encoding='utf-8')); print(s.get('project',{}).get('genre',''))")"

python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${WORKSPACE_ROOT}" \
  story-system "${CHAPTER_GOAL}" --genre "${GENRE}" --chapter {chapter_num} --persist --emit-runtime-contracts --format both
```

必备文件：`MASTER_SETTING.json`（调性/禁忌）、`volume_{NNN}.json`（卷级节奏）、`chapter_{NNN}.review.json`（必须节点/禁区）。缺失则阻断。

`chapter_{NNN}.json` 必须优先检查顶层 `chapter_directive`。`chapter_focus` 只能来自 `chapter_directive.goal` 或真实 query，不得从 `dynamic_context` 的参考摘要继承。

写作任务书排序必须固定为：
1. 本章硬性约束：`chapter_directive.goal/time_anchor/chapter_span/countdown/chapter_end_open_question`
2. CBN/CPNs/CEN 与 `must_cover_nodes`
3. 本章禁区：`forbidden_zones`，违反即不通过
4. 风格指引：reasoning、主角卡 OOC 警戒、anti_patterns
5. 场景写法补充：`dynamic_context`，仅作风格参考，不能覆盖章纲约束

### Step 1：context-agent 生成写作任务书

必须使用 `Agent` 工具调用 `context-agent`，不得由主流程自行整理任务书。

```text
Agent(
  subagent_type: "webnovel-writer:context-agent",
  prompt: "chapter={chapter_num}; project_root=${PROJECT_ROOT}; scripts_dir=${SCRIPTS_DIR}; storage_path=${PROJECT_ROOT}/.webnovel; state_file=${PROJECT_ROOT}/.webnovel/state.json（projection/read-model，仅兼容读取）。先 research，再按 本章硬性约束→CBN/CPNs/CEN→本章禁区→风格指引→dynamic_context补充参考 的顺序输出五段写作任务书。"
)
```

产物：一份写作任务书，能独立支撑 Step 2 起草。

### Step 2：起草正文

**根据 `${NOVEL_TYPE}` 选择起草策略：**

#### 策略 A：知乎短篇 (`zhihu-short`)

只根据任务书起草。加载 `style-reference`（如果项目存在该文件），将风格参考中的规则内化为写作时的主动判断。只输出纯正文，无占位符。中文思维写作。

**起草时的风格追问（必须在写作过程中执行）**：
每写完一个关键段落或对话，主动追问自己以下问题：
1. **这句为什么要这样写？** —— 这个句式/表达是否符合任务书第4段定义的角色性格和题材逻辑？
2. **如果换一种写法，效果会差在哪里？** —— 比如：如果用长句代替短句，是否会压慢节奏、削弱女主的直球感？如果用"缓缓说道"代替动作引导，是否会丧失画面感、与角色性格矛盾？
3. **这种写法的底层逻辑是什么？** —— 是角色性格决定表达风格？是题材阅读场景决定信息密度？是角色关系决定对话方式？

不要机械执行"不要缓缓""不要淡淡"等禁令——要理解"为什么不能缓缓"（因为当前角色的什么性格/当前题材的什么场景，使得缓缓会破坏代入感或节奏），并基于这种理解做出正确的写作选择。

如果任务书第4段提供了"风格推理"，必须以理解后的方式应用，不是背诵规则。

**字数与结构硬约束**：
- 严格遵守任务书中定义的字数目标。如果任务书要求"对标原文±10%"，则必须精确控制在该范围内。
- 禁止添加任务书/原文中没有的对话回合、情节细节或背景补充。每一句对话、每一个动作都必须服务于任务书定义的情节节点。
- 如果项目存在参考原文，起草时必须对标原文的信息密度和段落结构。原文一句话推进一个信息点，生成文本也必须一句话推进一个信息点；原文没有的细节，生成文本也不能添加。

#### 策略 B：网文长文 (`long-form`)

只根据任务书起草。不加载 core-constraints/anti-ai-guide（已内化到任务书）。只输出纯正文，无占位符。有结构化节点时围绕 CBN→CPNs→CEN 展开。中文思维写作。

### Step 3：审查

必须使用 `Agent` 工具调用 `reviewer-coordinator`，不得由主流程伪造审查 JSON。

`reviewer-coordinator` 会串行调用各维度 reviewer agent（setting / timeline / continuity / character / logic / ai_flavor / redundant_desc），汇总为统一 JSON。

```text
Agent(
  subagent_type: "webnovel-writer:reviewer-coordinator",
  prompt: "chapter={chapter_num}; chapter_file=${CHAPTER_FILE}; project_root=${PROJECT_ROOT}; scripts_dir=${SCRIPTS_DIR}; mode=default。严格输出 reviewer schema JSON，并保存到 ${PROJECT_ROOT}/.webnovel/tmp/review_results.json。"
)
```

**回退**：若 `reviewer-coordinator` 不可用，可切回单体 `reviewer`：

```text
Agent(
  subagent_type: "webnovel-writer:reviewer",
  prompt: "chapter={chapter_num}; chapter_file=${CHAPTER_FILE}; project_root=${PROJECT_ROOT}; scripts_dir=${SCRIPTS_DIR}。严格输出 reviewer schema JSON，并保存到 ${PROJECT_ROOT}/.webnovel/tmp/review_results.json。"
)
```

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" review-pipeline \
  --chapter {chapter_num} \
  --review-results "${PROJECT_ROOT}/.webnovel/tmp/review_results.json" \
  --metrics-out "${PROJECT_ROOT}/.webnovel/tmp/review_metrics.json" \
  --report-file "审查报告/第{chapter_num}章审查报告.md" \
  --save-metrics
```

blocking=true → 修复后重审，不进 Step 4。`--fast` 只启动 setting/timeline/continuity 三个维度 agent。`--minimal` 跳过。

### Step 4：润色

**根据 `${NOVEL_TYPE}` 加载对应参考：**

若 `NOVEL_TYPE=zhihu-short`：
- 加载 `shared/polish-guide-base.md` + `zhihu-short/polish-guide-zhihu.md`
- 加载 `shared/style-adapter-base.md` + `zhihu-short/style-adapter-zhihu.md`
- 加载 `zhihu-short/anti-ai-guide-zhihu.md`
- 加载 `typesetting.md`

若 `NOVEL_TYPE=long-form`：
- 加载 `shared/polish-guide-base.md`
- 加载 `shared/style-adapter-base.md`
- 加载 `shared/anti-ai-guide-base.md`
- 加载 `typesetting.md`

**向后兼容**：若 `shared/` 目录不存在，回退加载根目录的 `polish-guide.md`、`style-adapter.md`、`anti-ai-guide.md`。

顺序：修复非 blocking issue → 风格适配 → 排版 → Anti-AI 终检。

只改表达不改事实。`anti_ai_force_check=fail` 时不进 Step 5。`--minimal` 仅排版。

### Step 5：提交

#### 5.1 Data Agent 提取事实

必须使用 `Agent` 工具调用 `data-agent`，产出 fulfillment_result / disambiguation_result / extraction_result 三份 JSON，并复用 Step 3 的 review_results。

```text
Agent(
  subagent_type: "webnovel-writer:data-agent",
  prompt: "chapter={chapter_num}; chapter_file=${CHAPTER_FILE}; project_root=${PROJECT_ROOT}; scripts_dir=${SCRIPTS_DIR}。从正文提取事实，生成 .webnovel/tmp/ 下的 fulfillment_result.json、disambiguation_result.json、extraction_result.json；fulfillment_result.json 必须顶层包含 planned_nodes/covered_nodes/missed_nodes/extra_nodes；disambiguation_result.json 必须顶层包含 pending；extraction_result.json 必须严格按你的第7节格式输出顶层字段 accepted_events/state_deltas/entity_deltas/entities_appeared/scenes/summary_text，禁止包在 chapter/fulfillment/disambiguation/extraction 等外层对象里；accepted_events 子项必须包含 event_id/chapter/event_type/subject/payload；不直接写 state/index/summaries/memory。"
)
```

Data Agent 只提取事实+生成 artifacts，不直接写 state/index/summaries/memory。

#### 5.2 CHAPTER_COMMIT

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" chapter-commit \
  --chapter {chapter_num} \
  --review-result "${PROJECT_ROOT}/.webnovel/tmp/review_results.json" \
  --fulfillment-result "${PROJECT_ROOT}/.webnovel/tmp/fulfillment_result.json" \
  --disambiguation-result "${PROJECT_ROOT}/.webnovel/tmp/disambiguation_result.json" \
  --extraction-result "${PROJECT_ROOT}/.webnovel/tmp/extraction_result.json"
```

自动判定：blocking_count>0 或 missed_nodes 非空 或 pending 非空 → rejected，否则 accepted。

#### 5.3 验证投影

projection_status 五项（state/index/summary/memory/vector）全部 done 或 skipped。

chapter_status 由 projection writer 自动推进：accepted→committed，rejected→rejected。

#### 5.4 失败隔离

commit 未生成→重跑 5.2。projection 失败→只补跑失败项。不回退 Step 1-4。

### Step 6：Git 备份

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" backup \
  --chapter {chapter_num} \
  --chapter-title "{title}"
```

备份必须以解析后的 `PROJECT_ROOT` 为准，禁止从工作区父目录执行裸全量 Git add，避免把书项目仓库作为父仓库的嵌入仓库/submodule 加入。

## 充分性闸门

1. 正文文件存在且非空
2. 审查已落库（`--minimal` 除外）
3. blocking=true 必须停在 Step 3
4. anti_ai_force_check=pass（`--minimal` 除外）
5. accepted CHAPTER_COMMIT，projection 五项 done/skipped
6. chapter_status=committed（projection 自动推进）

## 失败恢复

审查缺失→重跑 Step 3。摘要/状态/记忆缺失→重跑 Step 5。润色失真→回 Step 4 修复后重跑 Step 5。
