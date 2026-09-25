# 学习路线：从会读架构图到能审视跨视角证据

建议分 8 次学习，每次约 60–120 分钟，可按基础调整。无需先读完全部论文才开始整理理解。下列问题是学习产物，不要求立即跑训练。

| 次序 | 阅读内容 | 这次只抓住什么 | 完成标志 |
|---|---|---|---|
| 1 | [入门指南](getting-started.md)；Diffusion Policy 图与方法 | observation、proprioception、action chunk、闭环；扩散怎么生成动作 | 画出一次“观察→预测→执行→重观察”的流程 |
| 2 | [OpenVLA](related-work/openvla.md)、[π0](related-work/pi0.md) | 离散动作 token 与连续 flow head 的区别 | 列出输入、骨干、输出、损失与推理步骤 |
| 3 | [FAST](related-work/fast-tokenizer.md)；回看动作序列 | 为什么逐维量化长动作块会很长，压缩如何改变序列 | 解释 FAST 与 Fast-WAM 的区别，区分训练速度/推理延迟 |
| 4 | [UVA](related-work/uva.md)、[WorldVLA](related-work/worldvla.md) | 视频和动作联合训练、交替预测、attention mask | 分清策略、forward dynamics、inverse dynamics |
| 5 | [Fast-WAM](related-work/fast-wam.md)，对照 [DreamZero](related-work/dreamzero.md) | 训练视频监督与测试想象的解耦；速度与成本 | 不看笔记说明 Fast-WAM 四种变体各改变什么 |
| 6 | [跨视角专题](cross-view-consistency.md) 与 [SCVC](related-work/selective-cross-view-consistency.md) | 同状态配对、动作坐标系、不变量、内插/外推 | 解释为何不同视角 RGB 不应直接相等，为什么必须有 paired control |
| 7 | [PAIWorld](related-work/paiworld.md)、[TriWorldBench](related-work/triworldbench.md)；选读 [3D-VLA](related-work/3d-vla.md) | 多视角生成的几何与视频评测，3D 中间表征 | 举一个“视频很好但动作失败”的可能例子，说明应测哪些量 |
| 8 | [评测指南](benchmarks.md)、[评测草案](evaluation.md)、[复现入口](reproduction.md) | 选择一个能真正检验问题的 baseline 和固定协议 | 写出可复现实验清单与资源需求，不急于提出创新点 |

## 新增：WAM深入阅读路线

适用于已理解基本VLA接口后，回应本轮“重点补有影响力WAM”的目标。每次围绕一个比较问题，可分多次完成；原来的8次入门路线仍可先走。

| 次序 | 配对阅读 | 核心问题 | 应写出的学习产物 |
|---|---|---|---|
| W1 | [选择依据](wam-influence-map.md)、[总综述](wam-survey.md) | 不同作者说的WAM是否同一种条件分布？ | 把policy、forward、inverse、joint各画一条信息流 |
| W2 | [UniPi](related-work/unipi.md)→[GR-1](related-work/gr1.md)→[GR-2](related-work/gr2.md) | 从视频计划到共享视频/动作模型，改变了什么？ | 列明独立IDM、共享参数、控制系统与缺失细节 |
| W3 | [GR-1](related-work/gr1.md)→[Seer](related-work/seer.md) | 视频只是辅助目标，还是进入动作计算路径？ | 画OBS/ACT与FRS/INV两套mask；解释3.31/3.41/3.64 |
| W4 | [UVA](related-work/uva.md)、[VideoVLA](related-work/videovla.md)、[WorldVLA](related-work/worldvla.md) | 连续扩散、mask建模、离散token各如何统一模态？ | 分别写训练loss与一次部署算法，不照搬同一个公式 |
| W5 | [Motus](related-work/motus.md)、[Cosmos Policy](related-work/cosmos-policy.md) | 条件/噪声开关怎样实现多个功能？ | 画Motus五模式与Cosmos三种训练条件；区分策略和规划 |
| W6 | [DreamZero](related-work/dreamzero.md)、[LingBot-VA](related-work/lingbot-va.md) | 联合生成、IDM、历史cache与真实反馈有何差别？ | 画同步/异步时间线，解释预测漂移和纠正 |
| W7 | [Fast-WAM](related-work/fast-wam.md)，回看VideoVLA/Motus | 怎样严格分离训练监督和测试生成？ | 列四变体改变项；解释为什么不同论文结论并不直接冲突 |
| W8 | [GE](related-work/genie-envisioner.md)、[Action Images](related-work/action-images.md)、[PAIWorld](related-work/paiworld.md) | 拼图、跨视图attention、几何投影、解码分别解决什么？ | 标明相机参数进入位置、哪些任务实际执行动作 |
| W9 | [V-JEPA 2](related-work/v-jepa2.md)、[鲁棒性研究](related-work/wam-robustness.md)、[SCVC](related-work/selective-cross-view-consistency.md) | 相机变化为什么影响动作，什么量应一致？ | 一页失败机制对照；分别列数据划分、坐标系、默认启用扰动 |
| W10 | [评测草案](evaluation.md)、[复现准备](reproduction.md) | 哪个公开baseline最先能验证研究问题？ | 冻结一个原始评测配置和一项相机扰动，列资源/版本缺口 |

如果只先精读六篇：**Fast-WAM→Seer→Cosmos Policy→LingBot-VA→GE→鲁棒性研究**；对动作几何特别感兴趣再接Action Images与SCVC。此顺序是学习组织，影响证据和方法局限见对应笔记。

## 每篇固定做一张卡片

1. 问题是什么，与用户关心的跨视角问题是否同一件事？
2. 训练和部署的输入分别是什么？有没有只有训练时可见的状态、深度或相机参数？
3. 图像/视频/语言/动作怎样编码？模块间谁能看到谁？
4. 数据来自哪里，动作坐标系与时间对齐是什么？
5. 每一项损失约束什么？有没有控制数据量、预训练或模型容量？
6. 输出是什么，测试时怎样采样、执行和重新观察？
7. 实验是离线视频、离线动作误差、仿真闭环、还是实机闭环？
8. 最强证据是哪张表，最重要局限是什么？

先用自己的话回答，再查原文；无法回答的项目写“待核实”，不要靠名称猜测。

## 后续研究工作节奏


- 学习阶段：完成上述卡片，确定研究的是策略视角泛化还是多视角生成几何一致性。
- 复现阶段：先跑通一个公开 baseline 的原始设置，核对数据中间过程和固定失败样本。
- 定义问题阶段：根据真实复现发现，提出可证伪的问题，写入 brainstorm.md。
- 实验阶段：固定 split、预算、对照、指标，执行一致性/数据配对/视角输入等消融。
- 写作阶段：保留失败结果、资源与局限；不能仅凭单一平均成功率或漂亮视频立论。

这些阶段有先后依赖。当前交付完成的是文献调研与复现准备，后续训练需要另行核实算力、目标机器人和数据权限。
