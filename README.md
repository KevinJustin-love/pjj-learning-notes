# PJJ Learning Notes

个人学习与研究笔记仓库，持续更新。目录组织参考 [Purshow_Notes](https://github.com/Purshow/Purshow_Notes)：按主题建立独立目录，每个目录提供 README 导航；正文保留 Markdown，配套 LaTeX、PDF、参考文献和图表。

资料冻结：2026-09-12。

## Contents

| 目录 | 内容 | 入口 |
|---|---|---|
| [embodied-rl-roadmap](./embodied-rl-roadmap/) | 具身强化学习路线图：经典控制、sim-to-real、真实世界 RL、离线 RL、Diffusion/VLA 后训练 | [README](./embodied-rl-roadmap/README.md) |
| [embodied-intelligence](./embodied-intelligence/) | 具身智能调研、VLA、世界模型、WAM 与相关论文 | [README](./embodied-intelligence/README.md) |
| [video-generation](./video-generation/) | 视频生成模型与世界模型阅读笔记 | [README](./video-generation/README.md) |
| [guides](./guides/) | Codex、Claude Code、AR 视频生成教程与快捷键 | [README](./guides/README.md) |
| [notes](./notes/) | 其他学习笔记目录索引 | [README](./notes/README.md) |

## Root files

根目录保留常用阅读副本与论文图表；完整的桌面 note 子目录结构打包在 [notes_archive.zip](./notes_archive.zip) 中。

## Reading order

1. 先读各主题目录的 README，了解范围与文件对应关系。
2. 阅读 Markdown 正文；需要打印或分享时使用 PDF。
3. LaTeX、BibTeX 和 figures 用于继续编译、修改和追溯来源。

## Compile

具身智能与视频生成笔记包含 LaTeX 源文件。建议使用 XeLaTeX 或 latexmk 编译；编译中间文件放在本地 build 目录，不提交生成缓存。
# 具身智能发展笔记

资料冻结：2026-09-12。

- `source/main.tex`：扩展版主讲义，约 12 页，含 RT-1/RT-2、π 系列、Code-as-Policy、Coding Is All You Need 补充、Cosmos、DreamDojo、DreamZero、VERA、OpenWAM、小米模型、GPT-6 Astra、RoboDojo 与 RoboWM-Bench。
- `source/technology_stack.tex`：VLA、WAM、Code-as-Policy 技术栈补充讲义。
- `source/references.bib`：论文与官方项目参考文献。
- `source/figures/`：从 arXiv 源码/HTML 提取的论文原图。
- `output/pdf/main.pdf`：扩展版主讲义 PDF。
- `output/pdf/technology_stack.pdf`：技术栈补充 PDF。
- `build/`：早期编译中间文件。

主讲义新增“Coding Is All You Need? Why We Need a World Model!”章节，讨论 Cosmos、DreamDojo、DreamZero、VERA 和可执行性评测。网页版本保持简洁，只保留主要路线和论文链接。



- output/pdf/main_labeled.pdf：论文缩写标签版 PDF（如 [RT-2]、[DreamDojo]、[OpenWAM]）。
