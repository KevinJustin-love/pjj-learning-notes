# FAST 动作 tokenizer 阅读笔记

核验日期：2026-09-25。论文：**FAST: Efficient Action Tokenization for Vision-Language-Action Models**，Karl Pertsch 等；固定阅读 [arXiv:2501.09747v1](https://arxiv.org/html/2501.09747v1)（2025-01-16）。**状态：读过算法、实验及附录，未复现。FAST tokenizer ≠ Fast-WAM。** 此处 FAST 是动作表示方法；π0-FAST 是使用它的策略；FAST+ 是预训练通用 tokenizer。

## 方法：把一段动作压缩成 token

逐时刻、逐维度分桶会在高频动作中产生大量重复 token。FAST 对一个动作块沿时间做 DCT（离散余弦变换），将结果缩放并取整，再按低频优先的顺序展开，用 BPE 压缩整数序列。解码时反向恢复，得到连续动作块。DCT 无需神经网络训练，BPE 词表是学习得到的部分；量化会带来误差，整体不是无损压缩。[论文 §IV–V、算法 1](https://arxiv.org/html/2501.09747v1#S5)

| 层级 | 输入 | 输出/训练目标 |
|---|---|---|
| FAST tokenizer | 已归一化的 `H × Da` 动作块，通常覆盖 1 秒 | 离散 token 序列；BPE 拟合动作系数序列 |
| VLA policy | 2–3 张 224×224 图像、任务文字、本体状态 | 自回归预测动作 token；next-token 交叉熵 |
| 解码与控制 | 预测 token，以及正确的时域/动作维度 | 逆 BPE、逆量化、逆 DCT、反归一化得到控制序列 |

官方 tokenizer 发布页提供 FAST+，拟合于 **1M 个真实机器人动作序列**，可独立于策略使用；支持 batch 编码/解码，建议输入先做分位数归一化。解码必须知道 `time_horizon` 和 `action_dim`，不能把一个 token 当作一个时刻的一个关节命令。[官方 FAST tokenizer 仓库/模型卡](https://huggingface.co/physical-intelligence/fast)

π0-FAST 去掉了连续 flow 解码路线，使用 VLM 自回归生成压缩动作 token。**FAST+ 的 1M 动作块数据与完整 π0-FAST 的策略预训练数据是两层不同训练，不能混用。** 官方 openpi 分别提供 π0 与 π0-FAST 的权重。[官方 openpi](https://github.com/Physical-Intelligence/openpi)

具体策略输入配置见论文附录 C；本体状态先分为 256 桶，再作为文本输入。这不矛盾：FAST 改善的是输出动作序列的预测目标，状态输入仍可采用普通分桶。[论文附录 C](https://arxiv.org/html/2501.09747v1#A0.SS3)

## 已核实数字与评测

**表 I：都是 1 秒动作块，按相近重建精度比较。**

| 数据 | 动作维数×控制频率 | 逐步分桶 token 数 | FAST 平均 token 数 | 压缩比 |
|---|---:|---:|---:|---:|
| DROID | 7×15Hz | 105 | 29 | 3.6 |
| T-shirt folding | 14×50Hz | 700 | 53 | 13.2 |

该表衡量表示长度，不能直接当作机器人成功率或端到端加速比。策略实验包含仿真与真实任务；不同任务分数也不同，如 LIBERO 为二元成功、table bussing 为正确分类物体的比例、laundry folding 按单件衣服是否完成统计。[论文表 I、§VI-A、附录 E](https://arxiv.org/html/2501.09747v1#S6)

训练效率与推理效率必须分开：论文 §VI-F 报告 π0-FAST 达到相近通用策略表现所需 GPU 小时约为 π0 的五分之一；**§VI-E 同时报 RTX 4090 上 π0-FAST 约 750ms/动作块，而连续 π0 通常在 100ms 内生成一块**。自回归仍需顺序解码多个 token，所以“FAST”不能解读成比 flow 策略推理更快。[论文 §VI-E–F](https://arxiv.org/html/2501.09747v1#S6.SS5)

本论文 LIBERO 将四个 suite 合并训练一个策略，而 OpenVLA 原论文各 suite 单独训练；即使测试名称相同，也不能直接把两篇的平均数作因果排名。FAST+ 训练混合还包含部分真实策略评测任务的数据；另行测试的未见数据集上的 tokenizer 压缩泛化，不等于所有策略测试任务都未见。[论文 §VI-C、附录 E](https://arxiv.org/html/2501.09747v1#S6.SS3)

## 成本、开放材料与限制

官方 FAST 发布在作者 Hugging Face 仓库，标注 Apache-2.0，提供 tokenizer 实现及词表；自定义 `.fit()` 通常为秒到分钟量级。**这只是 tokenizer 的拟合成本**，不包含训练一个数十亿参数 VLA。[官方模型卡](https://huggingface.co/physical-intelligence/fast)

完整 DROID 策略配方见附录 D：选择约 75k 条成功演示、过滤 idle actions，3 epochs/240k iterations、batch 256，约 8×H100 跑 4 天。这是特定策略训练预算，不是本机复现数据。[论文附录 D](https://arxiv.org/html/2501.09747v1#A0.SS4)

openpi 已公开 π0-FAST 和 DROID 适配 checkpoint，并把 DROID 训练说明称作原训练流程的近似开源实现。当前代码、公开权重和原实验配方可能有差异，复现需要固定版本。[官方 openpi 更新与 Model Checkpoints](https://github.com/Physical-Intelligence/openpi)

## 与跨视角一致性的关系：本项目分析

FAST 压缩的是时间序列动作，没有给图像增加三维一致性约束。DROID 配方训练时在两台外部相机中随机选一台，并使用腕部相机；不输入相机标定。这是视角多样性训练，不能据此宣称显式跨视角几何一致。[论文附录 D](https://arxiv.org/html/2501.09747v1#A0.SS4)

若研究跨视角一致性，可以固定 FAST 表示作为动作输出侧，以免换动作表示造成混淆；比较时同时固定 token 预算、动作块时长与重建精度。Fast-WAM 是否使用相似技术，应按 Fast-WAM 原文独立核实，名字中的“Fast”不能建立方法继承关系。

## 阅读顺序

图 3（分桶问题）→ 图 4/算法 1（DCT 与 BPE）→ 表 I（表示长度）→ §VI-E（推理成本）→ 附录 D/E（协议）。不要只读摘要中的训练加速数字。

## 本地原文复核

已与 [PDF 提取文本（论文入口）](https://arxiv.org/abs/2501.09747v1) 和源码交叉核对：[05_experiments.tex（论文入口）](https://arxiv.org/abs/2501.09747v1) 第 49/51 行为表 I 数字，第 163 行给出 750ms 与“100ms 内”，第 183 行是 5 倍 GPU 小时差异；[07_appendix.tex（论文入口）](https://arxiv.org/abs/2501.09747v1) 第 76 行明确为 8×H100、约 4 天、75k 演示、240k iterations；第 119 行确认四个 LIBERO suite 合并训练。依据本地源码，将延迟表述细化为“100ms 内”，其余核心数字一致。
