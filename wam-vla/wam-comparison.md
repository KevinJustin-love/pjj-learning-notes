# WAM 代表方法：架构、训练、评测与跨视角证据

检索核实日期：2026-09-25。本文保留首轮Fast-WAM/DreamZero/UVA/WorldVLA的紧凑对照。**扩展后的主入口为 [WAM深度综述](wam-survey.md)和[重点文献选择依据](wam-influence-map.md)**，已补UniPi、GR-1/2、Seer、VideoVLA、GE、Motus、Cosmos Policy、LingBot-VA等路线。所有结果是论文/官方仓库报告，未在本项目复现。版本固定在下表，仓库新增功能单独注明。

## 先建立几个区别

WAM在这些论文里并非严格统一的单一架构。DreamZero §2.2 将利用未来状态预测帮助动作学习的方法统称WAM；Fast-WAM允许训练预测未来、部署直接出动作；WorldVLA则混合显式动作条件前向模型与策略。读论文时逐项问：**预测目标是什么，未来预测是否以动作为条件，部署是否生成未来，生成结果是否真的参与选动作？** [DreamZero定义](https://arxiv.org/html/2602.15922v1#S2.SS2)、[Fast-WAM §3](https://arxiv.org/html/2603.16666v2#S3)、[WorldVLA §3](https://arxiv.org/html/2506.21539v1#S3)。

“视频/动作对齐”是两个模态表达同一行为；“时间一致性”是连续时刻状态合理；“跨视角一致性”要求同一时刻不同相机对同一3D状态的描述相容。这三个评测问题不能相互替代。下面“未报告”仅限已读论文的方法/实验，不声称排除了所有可能代码细节。

## 主线对照

| 论文与版本 | 输入→输出、骨干 | 训练监督与数据 | 部署时未来生成 | 跨视角证据 |
|---|---|---|---|---|
| [Fast-WAM](related-work/fast-wam.md)，2603.16666v2 | 当前多相机拼图+语言→32步动作；Wan2.2-5B+1B动作DiT | 动作/视频flow matching；LIBERO、RoboTwin、60h叠毛巾；无额外embodied预训练但有Wan预训练 | 否；首帧视频编码一次，动作仍10步去噪 | 拼图支持多相机；无显式跨视角损失/度量 |
| [DreamZero](related-work/dreamzero.md)，2602.15922v1 | 历史多相机拼图+语言+状态→视频和动作；Wan2.1-I2V-14B | 联合flow matching+teacher forcing；约500h AgiBot和DROID分别训练 | 是；视频按块自回归，动作与视频联合去噪；闭环用真观察更新cache | 拼图；video-action alignment有展示，非跨视角几何保证 |
| [UVA](related-work/uva.md)，2503.00200v3，RSS 2025 | 历史视频/动作+可选语言→共用潜表示→两个diffusion head；MAR-B，约0.5B | 动作/视频噪声MSE+mask多任务；公开仿真/UMI；先视频再联合 | Policy模式否；planner/forward dynamics模式可生成并选候选 | LIBERO仅第三人称；无跨相机一致性方法 |
| [WorldVLA](related-work/worldvla.md)，2506.21539v1 | 语言+历史帧→离散动作；图像+动作→下一图像；Chameleon | 目标动作/图像token交叉熵，图像权重0.04；LIBERO | 策略接口不需要；world模式可自回归下一帧 | “2帧”是历史帧；原版无显式跨视角机制 |

各行结构/训练证据分别见链接笔记中的§3/§III、训练设置及实验表；它们不具有统一数据、参数量、输入模态与训练预算，不能按单个成功率直接排序。

## 最值得记住的受控证据

1. **Fast-WAM的价值在于拆开训练与推理因素。** RoboTwin中无视频目标下降8.0个百分点；LIBERO中下降4.1，而直接策略与两种生成未来版本相差0.4–0.9个百分点。结论支持特定配方和任务中训练监督贡献较大；不能推成“显式规划从不必要”。[v2表1–2、§4.3](https://arxiv.org/html/2603.16666v2#S4.SS3)
2. **UVA是重要前例。** 2025年就把视频训练与动作推理解码分开；它还在BlockPush候选选择中使用视频前向预测，DP成功率38%→60%，真实模拟器75%。所以应分别研究“直接policy是否必须生成视频”和“多候选规划何时受益于预测”。[UVA §III-C、§VII](https://arxiv.org/html/2503.00200v3)
3. **DreamZero主要测更强分布变化。** AgiBot未见技能39.5%是任务进度，不是完整成功率；人类视频迁移后的结果使用目标技能演示继续训练。Fast-WAM当前目标任务训练与DreamZero任务零样本并非同一问题，二者没有被这里的数字互相证伪。[DreamZero §4–5](https://arxiv.org/html/2602.15922v1#S4)
4. **WorldVLA提醒区分world loss与其他结构改动。** 已有chunk+mask后再加world目标，只从76.6%升到78.1%；50帧FVD改善，10帧FVD却退步。应该逐项复现实验，不能把全部变化都归给“理解物理”。[WorldVLA表3–4](https://arxiv.org/html/2506.21539v1#S4.SS2)

## 速度数字必须附条件

| 方法 | 作者计时 | 条件/限制 |
|---|---|---|
| Fast-WAM v2 | 190ms，IDM810ms | 单5090D V2 32GB；32动作，动作10去噪步；当前代码有另一次加速更新 |
| DreamZero-Flash | 150ms、约7Hz重规划 | 2×GB200；table bussing任务进度74±10.1%，4步版83±6.1%；低层30Hz控制不等于7Hz推理 |
| UVA | 真机95ms；仿真0.23s | RTX3080 vs L40；16步动作；真机16去噪步；不同设置不可横向排榜 |
| WorldVLA | 表5报告FPS | 本文正文未给清楚计时GPU；不加入延迟排名 |

证据：[Fast-WAM §4.1/4.3.3](https://arxiv.org/html/2603.16666v2)、[DreamZero §3.2/6、表3](https://arxiv.org/html/2602.15922v1)、[UVA附录X-D](https://arxiv.org/html/2503.00200v3)、[WorldVLA表5](https://arxiv.org/html/2506.21539v1)。

## 开源复现选择

| 目标 | 可优先看的入口 | 当前核实到的限制 |
|---|---|---|
| 快速理解视频监督+直接策略 | [UVA官方仓库](https://github.com/ShuangLI59/unified_video_action) | 有PushT Colab/权重；建议至少4GPU训练；任务规模较小 |
| 研究有/无测试想象、LIBERO/RoboTwin | [Fast-WAM官方仓库](https://github.com/yuantianyuan01/FastWAM) | 代码/权重/处理数据已发布；8GPU LIBERO、64GPU RoboTwin为作者训练配置；新Optional-IDM不属于论文v2 |
| 大视频骨干的任务/环境泛化 | [DreamZero官方仓库](https://github.com/dreamzero0/dreamzero) | 训练/微调和DROID资源已发布；默认至少2GPU推理；README公开路径0.6s/3s与论文150ms有差别 |
| 离散统一模型的机制消融 | [WorldVLA旧入口](https://github.com/alibaba-damo-academy/WorldVLA) | 已重定向RynnVLA-002；必须固定原版历史提交/权重，不能用现main升级结果冒充旧论文 |

建议顺序是阅读Fast-WAM的问题设置→UVA的实现前例→DreamZero的泛化设置→WorldVLA的离散替代。实际开跑先以一个现成checkpoint复核其原论文指标；这是根据公开资产完整性给出的执行建议，不是本项目已完成实验。

## 名称去混淆：UVA与两个UniVLA

| 名称 | 正确论文与ID | 官方入口 |
|---|---|---|
| UVA | Unified Video Action Model，2503.00200 | [ShuangLI59/unified_video_action](https://github.com/ShuangLI59/unified_video_action) |
| UniVLA（BAAI） | Unified Vision-Language-Action Model，2506.19850 | [baaivision/UniVLA](https://github.com/baaivision/UniVLA)；Emu3统一离散建模/视频world训练，官方仓库标ICLR2026 |
| UniVLA（OpenDriveLab） | UniVLA: Learning to Act Anywhere with Task-centric Latent Actions，2505.06111 | [OpenDriveLab/UniVLA](https://github.com/OpenDriveLab/UniVLA)；任务相关latent action路线，RSS2025 |

后两项本轮仅核对论文、作者/官方repo和路线，没有进行与四篇同深度的方法实验审计；不得把两家的指标、架构与权重交叉引用。[BAAI论文](https://arxiv.org/abs/2506.19850)、[OpenDriveLab论文](https://arxiv.org/abs/2505.06111)。

## 对跨视角课题的直接约束

这四篇中，Fast-WAM和DreamZero的多相机拼图提供了可改造入口，但没有证明跨视角一致性已建立。WorldVLA原版的时间多帧与UVA的单第三人称也不能补足此证据。**由本文方法和实验范围推断**，若目标是跨视角一致性，需要额外定义：同一世界状态下相机变化时动作是否稳定、不同视角预测能否对应同一物体轨迹、几何约束改善是否能转化为成功率。此处是待检验问题；具体跨视角方法与几何指标应由专门文献支持，不能仅依据WAM标题或示例视频下结论。
