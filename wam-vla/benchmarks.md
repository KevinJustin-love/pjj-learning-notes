# 怎样读懂 WAM / VLA 的评测

核查日期：2026-09-25。本页解释基准用途；没有运行任何模型，也不把不同论文的数字合并排名。

## 基准地图

| 基准/设置 | 测什么 | 看结果时必须问什么 | 官方来源 |
|---|---|---|---|
| LIBERO | 机器人操作、知识迁移；后续工作也广泛使用多任务训练设置 | Spatial/Object/Goal/10 还是原始 lifelong setting？训练数据、任务数、init states、每任务次数是否一致？ | [论文](https://arxiv.org/abs/2306.03310)、[仓库](https://github.com/Lifelong-Robot-Learning/LIBERO) |
| LIBERO-Plus | 对布局、相机、初始机器人状态、语言、光照、背景、噪声等扰动的鲁棒性 | 是否只报告 Camera 子集？是否训练已覆盖测试视角？腕部相机动不动？ | [CVPR 2026](https://openaccess.thecvf.com/content/CVPR2026/html/Fei_LIBERO-Plus_A_Progressive_Robustness_Benchmark_for_Visual-Language-Action_Models_CVPR_2026_paper.html)、[仓库](https://github.com/sylvestf/LIBERO-plus) |
| RoboTwin 2.0 | 双臂任务，合成数据与域随机化 | clean/random 对应何种配置？任务版本、数据量、是否包含相机扰动？ | [论文](https://arxiv.org/abs/2506.18088)、[官方文档](https://robotwin-platform.github.io/doc/) |
| CALVIN | 语言条件长时序操作 | ABC→D 还是 ABCD→D？平均连续完成任务数还是单个任务成功率？是否闭环？ | [论文](https://arxiv.org/abs/2112.03227)、[仓库](https://github.com/mees/calvin) |
| TriWorldBench | 头部、左腕、右腕同步未来视频的一致性 | 模型是否动作条件化？视频时长、分辨率、指标评估器与聚合权重是否相同？ | [论文](https://arxiv.org/abs/2609.26314)、[本地笔记](related-work/triworldbench.md) |

LIBERO 原始论文定义 Spatial/Object/Goal/100 共 130 个任务；100 再划为 90/10。如今常见四套平均往往是 Spatial/Object/Goal/10 共 40 任务，不能看到“四套”就认为等于原始 130 任务。[LIBERO 官方说明](https://github.com/Lifelong-Robot-Learning/LIBERO)。

LIBERO-Plus 的会议版标题为 *A Progressive Robustness Benchmark for Visual-Language-Action Models*；早期 arXiv 标题为 *In-depth Robustness Analysis...*。引用时核对版本。其官方评测实现的变体任务组织与常见 LIBERO 每任务 50 次不同，不能机械套用固定次数。[会议版](https://openaccess.thecvf.com/content/CVPR2026/html/Fei_LIBERO-Plus_A_Progressive_Robustness_Benchmark_for_Visual-Language-Action_Models_CVPR_2026_paper.html)、[评测说明](https://github.com/sylvestf/LIBERO-plus)。

## 四类指标不能互相替代

| 层次 | 指标例子 | 能支持的结论 | 盲点 |
|---|---|---|---|
| 离线动作预测 | MSE、方向/姿态误差、夹爪准确率 | 对示范动作的拟合 | 多解动作会受罚；不能测错误累积和恢复 |
| 生成视频质量 | PSNR、SSIM、LPIPS、FVD | 像素/感知/分布质量的一部分 | 看起来合理不等于服从动作、物理正确或跨视角一致 |
| 几何/跨视角 | 有标定时重投影、对应点误差、三维一致性；TriWorldBench 子项 | 各视角能否解释同一状态或行为 | 深度/匹配/VLM 评估器有误差；遮挡需要可见性处理 |
| 闭环控制 | success rate、task progress、长任务链完成长度 | 在给定环境、重规划与终止规则下真正完成任务的能力 | 不能跨任务难度、硬件、数据规模直接排名 |

例如，PAIWorld 的主要证据在生成视频与几何代理指标；SCVC 的重点在 held-out viewpoint 的仿真控制；不能用前者的视频指标证明后者的动作提升。[PAIWorld](related-work/paiworld.md)、[SCVC](related-work/selective-cross-view-consistency.md)。

## 视角泛化需要明确哪一种“没见过”

- **分布内**：训练与测试相机分布相同，用于确认基本能力。
- **内插**：训练视角范围内部留出部分相机，测试在这些位置。
- **外推**：测试超出训练方位、俯仰或距离范围。
- **观测变化**：仅外部相机改变、腕部相机改变、相机缺失，难度与信息量都不同。

训练时见过几乎一样的场景或同一演示的另一相机帧，会让结论含混。应记录 trajectory/state/scene/camera ID，先划分状态与轨迹，再在各自 split 内生成配对；相机角度集合也按目标独立留出。若研究只针对同任务同场景的相机变化，应明确范围，不宣称新场景泛化。这是本项目拟采用的实验纪律，具体已有论文协议见 [SCVC](related-work/selective-cross-view-consistency.md)。

## 读速度表时的核对表

同时记录 GPU 型号/数量、精度、batch size、图像大小、视角数、视频帧数、动作块长度、去噪步数、编译/KV cache、是否异步执行、预处理/传输/控制开销和计时起终点。首次编译时间与稳定运行延迟分别报告。

Fast-WAM 的 190ms 与 DreamZero 的约 7Hz 使用不同硬件和算法配置，不能直接判断谁更高效；生成一段50步动作也不代表50次独立反馈决策。[Fast-WAM](related-work/fast-wam.md)、[DreamZero](related-work/dreamzero.md)。

## 支撑论文的最低证据组合

这是阅读与实验设计建议：保持原始设置的 baseline；数据量与训练预算匹配的 control；分布内与 held-out 视角结果；逐任务/逐视角结果和不确定性；固定失败样本；完整计算成本。若主张的是“生成几何改善控制”，还需要把视频/几何改善和闭环控制改善连接起来的消融，不能仅分别展示两张好看的表。
## 本轮补充：WAM评测入口

新增[WAM/VLA鲁棒性研究笔记](related-work/wam-robustness.md)，覆盖LIBERO-Plus与RoboTwin2.0-Plus。RoboTwin协议为50任务×8配置×50 episodes；Camera默认C1距离与C3朝向，只变头部相机，C2球面位置关闭。不要将其直接等同于任意相机位姿外推。

各新论文的指标差异见[WAM深度综述](wam-survey.md)：GR-1/2与Seer的CALVIN平均链长、Cosmos/Motus/DreamZero的部分完成分、UniPi的视频分类器分数、Action Images开环SR均需保持原定义。原表Total、最佳checkpoint选择、基线重实现与计时脚注都在逐篇notes保留。
