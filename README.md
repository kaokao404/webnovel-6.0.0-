# Webnovel 6.0 — AI 辅助网文创作工作流

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](./LICENSE)

一套面向网文作者的系统化 AI 创作工作流，基于 Claude Code 的 Skill 架构，覆盖从灵感孵化到章节产出的完整创作链路。

> **核心理念**：用结构化约束释放创作自由，让 AI 成为可靠的写作搭档而非替代品。

---

## ✨ 功能概览

本系统包含 7 个核心技能模块，形成完整的创作闭环：

| 模块 | 指令 | 功能 |
|------|------|------|
| 🚀 **项目初始化** | `/webnovel-init` | 深度访谈式收集创作意图，生成项目骨架、世界观约束与角色档案 |
| 📋 **大纲规划** | `/webnovel-plan` | 基于总纲生成卷纲、时间线与章纲，支持增量迭代 |
| ✍️ **章节写作** | `/webnovel-write` | 上下文感知起草 → 自审 → 润色 → 提交的标准化流程 |
| 🔍 **质量审查** | `/webnovel-review` | 多维度评估章节质量，输出可执行的改进建议 |
| ❓ **信息查询** | `/webnovel-query` | 快速检索设定集、角色状态、伏笔分布与金手指进度 |
| 🧠 **模式学习** | `/webnovel-learn` | 从成功创作中提取模式，持续优化项目记忆库 |
| 📊 **管理面板** | `/webnovel-dashboard` | 只读项目总览，查看实体关系图谱与章节进度 |

---

## 🚀 快速开始

### 环境要求

- [Claude Code](https://claude.ai/code) 已安装并登录
- GitHub CLI (`gh`) 已配置

### 安装

```bash
# 克隆本仓库到 Claude Code 的 skills 目录
git clone https://github.com/kaokao404/webnovel-6.0.0-.git

# 将 skills/ 目录下的所有模块注册到 Claude Code 中
# （按照 Claude Code 的 skill 加载机制，通常位于 ~/.claude/skills/ 或项目级 .claude/skills/）
```

### 启动创作

```bash
# 1. 初始化新项目
/webnovel-init

# 2. 规划大纲
/webnovel-plan

# 3. 开始写作
/webnovel-write

# 4. 审查质量
/webnovel-review
```

---

## 📁 项目结构

```
webnovel-6.0.0/
├── skills/
│   ├── webnovel-init/          # 项目初始化
│   │   ├── SKILL.md
│   │   └── references/         # 创意库、世界观模板、市场分析
│   ├── webnovel-plan/          # 大纲规划
│   │   ├── SKILL.md
│   │   └── references/         # 情节框架、节奏控制、卷纲模板
│   ├── webnovel-write/         # 章节写作
│   │   ├── SKILL.md
│   │   ├── evals/              # 写作评估标准
│   │   └── references/         # 写作技法、反 AI 检测、风格适配
│   ├── webnovel-review/        # 质量审查
│   │   ├── SKILL.md
│   │   ├── evals/              # 审查指标
│   │   └── references/         # 常见问题、节奏控制
│   ├── webnovel-query/         # 信息查询
│   │   ├── SKILL.md
│   │   └── references/         # 查询规范、伏笔追踪
│   ├── webnovel-learn/         # 模式学习
│   │   └── SKILL.md
│   └── webnovel-dashboard/     # 管理面板
│       └── SKILL.md
└── LICENSE
```

---

## 🔄 创作工作流

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  webnovel   │───→│  webnovel   │───→│  webnovel   │
│   -init     │    │   -plan     │    │   -write    │
│  (立项)     │    │  (规划)     │    │  (写作)     │
└─────────────┘    └─────────────┘    └──────┬──────┘
                                             │
                         ┌───────────────────┼───────────────────┐
                         ▼                   ▼                   ▼
                  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
                  │  webnovel   │    │  webnovel   │    │  webnovel   │
                  │   -query    │←──→│  -review    │    │   -learn    │
                  │  (查询)     │    │  (审查)     │    │  (进化)     │
                  └─────────────┘    └─────────────┘    └─────────────┘
```

### 典型迭代流程

1. **`/webnovel-init`** — 确定题材、世界观、核心卖点、主角人设
2. **`/webnovel-plan`** — 生成总纲 → 卷纲 → 章纲，建立时间线
3. **`/webnovel-write`** — 按章纲写作，自动检索设定保持自洽
4. **`/webnovel-review`** — 评估节奏、爽点、人物一致性
5. **`/webnovel-query`** — 写作中随时查询伏笔、角色状态、势力关系
6. **`/webnovel-learn`** — 总结本轮成功模式，优化后续创作

---

## 🛠️ 技术特性

- **上下文感知**：每个技能都能读取项目记忆库，确保设定不自相矛盾
- **增量迭代**：支持在现有大纲/章节基础上持续追加，不重做全局
- **反 AI 检测**：内置反 AI 化写作指南，输出更接近真人风格
- **多题材适配**：覆盖玄幻、都市、悬疑、游戏等主流网文品类
- **知乎短篇支持**：专门适配知乎体短篇小说的写作与润色模式

---

## 📜 许可证

[GNU General Public License v3.0](./LICENSE)

---

## 🤝 贡献

欢迎提交 Issue 和 PR，共同完善网文创作工作流。

---

> *"写作是孤独的，但创作不必是。"*
