# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Type

This is an **Obsidian vault** — a personal knowledge management system. Content lives in markdown files organized by topic.

## Project Structure

```
note_obsidian/
├── DataCollection/     # 数据采集相关笔记
├── DB/                 # 数据库相关笔记
├── os/                 # 操作系统相关笔记
├── DayPlan/            # 日常计划（非笔记）
├── templates/          # 笔记模板
├── .obsidian/          # Obsidian 配置（忽略）
├── .trae/              # Trae IDE 配置
├── .claude/            # Claude Code 配置
└── README.md           # 笔记导航索引
```

## Project Rules

The `.trae/rules/project_rules.md` defines formatting standards:

- **Markdown 标题**: 使用 `# / ## / ###`，标题之间不空行，标题与正文空一行
- **代码块**: 前后空一行，上方写说明
- **列表**: 有序列表用于步骤，无序列表用于要点，列表项不空行
- **强调**: 行内代码用反引号，重点/定义用加粗
- **分隔**: 使用 `---` 分隔章节
- **文件命名**: 中文命名，文件名 = 一级标题

## README Navigation

The `README.md` serves as the index for all notes. The `.trae/skills/update/SKILL.md` defines a skill that:

1. Scans note folders (excluding `.obsidian`, `.trae`, `DayPlan`, `templates`)
2. Extracts titles and key sections from each note
3. Updates README.md with a table of contents

When adding new notes, follow the project rules and update README.md accordingly.
