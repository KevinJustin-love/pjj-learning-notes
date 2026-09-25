# PJJ Learning Notes

个人学习与研究笔记仓库。按主题阅读，每个目录的 README 提供正文入口。

## 主题索引

| 主题 | 内容 | 推荐入口 |
|---|---|---|
| [WAM / VLA](wam-vla/README.md) | 24篇论文笔记；模型架构、训练、评测与跨视角一致性 | [深度综述](wam-vla/wam-survey.md) · [论文索引](wam-vla/related-work/README.md) · [学习路线](wam-vla/reading-plan.md) |
| [具身强化学习](embodied-rl-roadmap/README.md) | 经典控制、仿真迁移、真实世界RL、离线RL与VLA后训练 | [路线图](embodied-rl-roadmap/embodied_rl_roadmap.md) |
| [视频生成](video-generation/README.md) | 视频生成发展、世界模型与自回归视频生成 | [发展笔记](video-generation/video_generation_notes.md) · [AR教程](video-generation/AR_Video_Gen_Tutorial_revised.md) |
| [Codex 与 Claude Code](codex-claude-code/README.md) | AI编程工具使用指南与编辑器快捷键 | [工具目录](codex-claude-code/README.md) |
| [历史归档](archives/README.md) | 尚未拆分到主题目录的原始学习资料 | [原始压缩包](archives/notes_archive.zip) |

## 目录约定

- `README.md`：主题介绍与阅读导航。
- `*.md`：主要阅读版本；`related-work/`：按论文组织的调研笔记。
- `assets/`：已有主题的图片与图表，按 `original/`、`rendered/` 等子目录整理。
- `*.tex`、`*.bib`、`*.pdf`：已有主题的可编辑或打印版本。
- `archives/`：保留历史原始资料，不与当前主题笔记混放。

`wam-vla/` 只保存Markdown笔记，文献通过名称、版本与原文链接引用。

## 阅读顺序

1. 从本页选择主题目录。
2. 阅读该目录 README，再阅读 Markdown 正文。
3. WAM/VLA从入门指南或总综述进入逐篇笔记；其他主题可按需阅读其PDF或LaTeX版本。

## 个人笔记页面

简版索引也发布在 [kevinjustin-love.github.io/notes.html](https://kevinjustin-love.github.io/notes.html)，适合快速浏览；本仓库保留完整源文件、参考文献和配套资源。

## 更新记录

- 2026-09-25：加入WAM/VLA笔记；移除仅含占位说明的 `notes/`，将历史压缩包移至 `archives/`，统一主题阅读入口。
- 各笔记的资料核查日期、论文版本与未复现边界以正文为准。
