# WAM / VLA：架构、训练、评测与跨视角一致性

面向初学者的中文调研与学习笔记。更新于 **2026-09-25**，整理24篇论文，重点关注 World Action Model、Vision-Language-Action 和跨视角一致性。

## 从哪里开始

1. [入门指南](getting-started.md)：观测、动作、策略、世界模型与闭环。
2. [WAM深度综述](wam-survey.md)：技术谱系、信息流、训练、推理、关键消融和评测口径。
3. [重点论文选择依据](wam-influence-map.md)：已有后续采用证据与新近相关候选。
4. [跨视角专题](cross-view-consistency.md)：多相机输入、几何一致性、动作一致性与视角泛化。
5. [学习路线](reading-plan.md)：8次入门学习与10次WAM深入阅读。
6. [24篇论文索引](related-work/README.md)：论文完整名称、固定版本、原文链接和逐篇精读。

## 重点方法

| 阅读目的 | 入口 |
|---|---|
| 视频监督与测试时想象 | [Fast-WAM](related-work/fast-wam.md)、[UVA](related-work/uva.md) |
| 代表性WAM架构 | [Cosmos Policy](related-work/cosmos-policy.md)、[LingBot-VA](related-work/lingbot-va.md)、[Motus](related-work/motus.md)、[DreamZero](related-work/dreamzero.md) |
| 技术前身与未来表征 | [UniPi](related-work/unipi.md)、[GR-1](related-work/gr1.md)、[GR-2](related-work/gr2.md)、[Seer](related-work/seer.md) |
| 多视角与控制 | [Genie Envisioner](related-work/genie-envisioner.md)、[Action Images](related-work/action-images.md)、[SCVC](related-work/selective-cross-view-consistency.md) |
| 相机失败与评测 | [V-JEPA 2](related-work/v-jepa2.md)、[WAM鲁棒性研究](related-work/wam-robustness.md)、[TriWorldBench](related-work/triworldbench.md) |

## 其他笔记

- [VLA基础对照](vla-foundations.md)：Diffusion Policy、OpenVLA、π0与FAST。
- [四篇WAM紧凑对照](wam-comparison.md)：Fast-WAM、DreamZero、UVA、WorldVLA。
- [评测与指标指南](benchmarks.md)、[跨视角评测协议草案](evaluation.md)。
- [数据资源](datasets.md)、[复现准备](reproduction.md)、[复现记录模板](reproduction-template.md)。

## 内容与证据

逐篇笔记记录架构、训练目标与数据、推理步骤、实验设置、消融、资源成本、开源状态及跨视角局限。论文原文的矛盾、未披露参数和未验证的推断保留明确标记。

此目录只发布Markdown笔记。文献原件通过论文名称及arXiv链接访问；笔记里的章节、表号和TeX文件名用于定位原文，PDF/源码引用已改为在线原文入口。这里不包含论文PDF、TeX包、模型权重或训练数据。

结果均为作者报告，未运行模型复现。生成质量、部分完成分、开环/闭环成功率及跨视角几何指标分别记录；不把不同协议的数值拼成排行榜。
