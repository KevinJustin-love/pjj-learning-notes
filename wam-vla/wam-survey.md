# WAM 深度综述：架构、训练、评测与跨视角问题

更新：2026-09-25。面向第一次学习 WAM 的读者。本文配套24篇论文阅读笔记。下文不是统一实验排行榜，所有数值均为对应作者报告，本项目尚未复现。

阅读入口：[选择依据与影响证据](wam-influence-map.md) → 本文 → 逐篇笔记与在线原文 → [跨视角专题](cross-view-consistency.md)。若时间有限，先精读 **Fast-WAM、Cosmos Policy、LingBot-VA、Motus、DreamZero**；补 **GR-1→Seer** 理解预测表征，再读 **GE、Action Images、鲁棒性研究** 对接跨视角课题。

## 1. 先用四个条件分布认识 WAM

记当前/历史观察为 `o`，未来观察为 `o+`，语言为 `l`，动作块为 `a`。下面是理解工具，不意味着每篇论文都显式建模四个分布：

| 模块 | 学什么 | 在线用法 |
|---|---|---|
| Policy | `p(a | o,l)` | 直接产生动作 |
| Forward dynamics / world model | `p(o+ | o,a)` | 预测一个候选动作的后果 |
| Inverse dynamics | `p(a | o,o+)` | 从希望发生的转移恢复动作 |
| Joint model | `p(a,o+ | o,l)` | 联合产生动作和对应未来 |

**WAM目前没有统一严格的架构定义。** 训练时预测未来、部署直接行动；在线生成未来再反推动作；模拟多个候选再规划，都有人置于这条研究脉络。判断一篇论文，首先画清信息依赖：动作是否读取未来表示？未来是否以候选动作为条件？未来是否在部署时计算？模型是否真正比较候选？学习世界模型和使用它规划是两件需要分别证实的事。[Fast-WAM](related-work/fast-wam.md)、[Cosmos Policy](related-work/cosmos-policy.md)、[V-JEPA 2](related-work/v-jepa2.md)恰好给出不同答案。

## 2. 架构谱系：哪些模块在交换信息

### A. 视频计划与预测表征

| 论文 | 关键计算图 | 训练信号 | 部署未来的角色 |
|---|---|---|---|
| [UniPi](related-work/unipi.md) | 文本+首帧→视频扩散→独立IDM→动作 | 视频去噪；IDM动作MSE，分开训练 | 必須完整生成视频计划；原实验开环 |
| [GR-1](related-work/gr1.md) | 冻结MAE/CLIP→GPT；OBS/ACT查询各自读取真实历史 | 像素MSE+臂Smooth-L1+夹爪BCE | 预测查询彼此不可见，动作不读取已生成图像 |
| [GR-2](related-work/gr2.md) | VQGAN离散视觉码→GPT；cVAE生成连续动作块 | 视频预训练→机器人联合学习；细节披露有限 | §3.4描述据预测视觉轨迹推动作；具体token依赖与mask未披露 |
| [Seer](related-work/seer.md) | 历史→FRS未来表征→INV动作；单向attention | 0.5视觉MSE+臂Smooth-L1+0.01夹爪BCE | 在线需要预测表征；不必先解码RGB或跑视频扩散 |

这组能解释重要区别：GR-1 的未来损失训练共享参数；Seer 还让动作显式读取预测未来表示。它们不应都被画成“先生成视频，再经IDM”的同一条流水线。

### B. 统一视频—动作模型

| 论文 | 表示与骨干 | 训练方式 | 动作与未来的关系 |
|---|---|---|---|
| [UVA](related-work/uva.md) | MAR-B共享mask建模；连续视频/动作潜表示；两个diffusion head | 两阶段视频→联合；噪声MSE | 根据mask支持直接策略、前向/逆动力学与规划；policy仅跑动作head |
| [WorldVLA](related-work/worldvla.md) | Chameleon；图像VQ token、逐维动作bins | action CE+0.04 image CE | 两种任务混合；策略无需生成图像；动作chunk使用特殊mask |
| [VideoVLA](related-work/videovla.md) | CogVideoX-5B；视频latent+连续动作token | DDPM共同去噪，共享noise schedule | 双向跨模态attention；部署仍预测未来latent |
| [Motus](related-work/motus.md) | Wan2.2-5B+Qwen3-VL+动作专家的MoT | 双模态flow matching；独立noise levels | 五模式切换；Joint同时生成，VLA模式仍保留噪声视频输入 |
| [DreamZero](related-work/dreamzero.md) | Wan2.1-I2V-14B；视频/动作分块自回归 | 联合flow matching、真实历史teacher forcing | chunk内联合、chunk间自回归；真观察更新历史 |
| [Fast-WAM](related-work/fast-wam.md) | Wan2.2-5B+1B动作DiT | 视频/动作flow matching，专门设计mask | 训练保留视频目标；部署仅首帧编码一次+动作去噪 |

本表的 **CE、DDPM、flow matching不是可随意互换的写法**。噪声时间方向、mask与模态能否互读决定“把视频分支删掉”是否成立；不能仅凭相同损失名称认定各方法等价。

### C. 当前骨干适配、条件生成与相邻世界模型

| 论文 | 特殊设计 | 训练/推理关键点 | 与跨视角的联系 |
|---|---|---|---|
| [Cosmos Policy](related-work/cosmos-policy.md) | Cosmos Predict2-2B内将状态、动作、value注入latent frames | EDM；策略/前向模型/value三种条件mask；可用额外rollout训练规划 | 多相机latent输入/预测，未给显式几何一致性目标 |
| [LingBot-VA](related-work/lingbot-va.md) | Wan视频专家+动作专家；因果历史+IDM | 先未来视觉后动作；v2以FDM连接异步控制与真实反馈 | 多视角latent拼接，时序反馈不等于相机几何一致 |
| [GE / GE-Act](related-work/genie-envisioner.md) | LTX-Video多视图attention+160M动作DiT | 视频领域预训练→动作学习→任务适配；单次视觉前向缓存+动作去噪 | 明确跨视图特征通信；GE-Sim另有依赖相机参数的Pose2Image |
| [V-JEPA 2-AC](related-work/v-jepa2.md) | 冻结视频encoder+300M动作条件latent predictor | latent L1+rollout；CEM搜索，图像目标MPC | 明确报告相机位置引发动作坐标轴误差 |
| [Action Images](related-work/action-images.md) | 动作姿态投影为多视图RGB热图，与观察统一建模 | masked flow matching；多视图几何解码；骨干型号原文有冲突 | 显式相机条件/三维解码；主控制实验开环 |

本节是对逐篇原文的压缩；数据、mask细节、章节位置、版本冲突和资产见每行笔记。V-JEPA 2-AC是邻近的latent planning路线，Action Images是新近直接相关候选，不能只因同列此表就认为影响力与成熟度相同。

## 3. 如何读训练：先分清四种“预训练”

1. **通用基础模型预训练**：Wan、CogVideoX、Cosmos、Chameleon、VLM/MAE已见过什么数据。Fast-WAM“无额外embodied pretraining”仍然继承Wan先验。
2. **机器人领域视频适配**：如GE-Base、Motus Stage1，将一般视频模型转向机器人场景。
3. **带动作或潜动作的跨任务预训练**：如LingBot 16k小时机器人数据、Motus弱监督光流潜动作、Seer DROID。
4. **目标平台/任务微调**：LIBERO、RoboTwin或自采任务。某论文目标域仅用几十演示，不表示整个模型只看过几十条数据。

### 可对照的训练成本与缺口

| 方法 | 已核实的训练单位/预算 | 不能省略的条件 |
|---|---|---|
| UniPi | 单个视频模型2M steps、batch2048、256 TPU-v4 | 未给总时长；还有时域/空间超分和上游视频预训练 |
| GR-1 | 195M总/46M可训练；视频50 epochs，CALVIN20 epochs | 冻结MAE/CLIP；没有完整GPU-hours |
| Seer | 仿真8×4090；CALVIN预训40h+微调24h，LIBERO30h+6h | 不含可由此推算的DROID全成本 |
| VideoVLA | 32×MI300X，100k预训+15k微调，batch256 | 原文无足够总时长；使用22.5M帧OXE子集 |
| GE | Base 32×A100约7+3天；动作16×A100约3天 | 后续任务视频/策略适配另计 |
| Motus | 三阶段约8000/10000/400 GPU-hours | 未明确型号；不含Wan/Qwen上游成本 |
| Cosmos Policy | LIBERO64×H100×48h；RoboCasa32×H100×48h；ALOHA8×H100×48h | 三个独立任务配置；规划另收rollout再训练 |
| LingBot-VA | 16k小时机器人数据、1.4T tokens | 数据小时不等于GPU小时，完整训练硬件时间未披露 |
| V-JEPA 2-AC | <62h DROID；batch256，94.5k更新 | 冻结大视频encoder；另有百万小时级通用视频预训练 |
| Action Images | 附录32×A100，100k步、每卡batch1 | 正文/附录骨干14B与5B不一致，不能当作已闭合配方 |

未列方法的具体配置仍在各自notes中；不根据参数量猜训练显存，也不把作者多卡配方写成我们现有机器可以承担的预算。论文没给的字段明确记缺口比补一个近似值更有用。

## 4. 核心争论：视频到底在什么时候起作用

下表选的是同论文的对照，避免用不同论文主表的分差替代消融。

| 要回答的问题 | 更有解释力的原文证据 | 解释边界 |
|---|---|---|
| 视频辅助目标有用吗？ | GR-1 ABCD→D链长3.33→3.82；再加视频预训练→4.21 | 前一步是训练目标，后一步是预训练；不证明跨视角机制 |
| 动作读取未来表征有用吗？ | Seer从头训练：BC3.31，独立未来预测3.41，未来条件动作3.64 | 一种小型GPT配方；不能据此断言必须生成完整视频 |
| 大视频先验是否重要？ | VideoVLA相同架构从头训12.6，CogVideoX初始化80.4 | SIMPLER三个任务组；不是跨全部任务的通用倍数 |
| 去掉视频损失就等于去掉推理想象吗？ | VideoVLA无视频loss/仅动作两种消融都改变训练路径 | 没有单独隔离部署未来生成的贡献 |
| 专门对齐后能否省部署未来？ | Fast-WAM四变体：RoboTwin Fast91.8、Joint90.6、IDM91.3、无视频83.8 | 单一受控任务配方；不能推导所有任务不需要规划 |
| 联合生成永远无收益吗？ | Motus Random：Joint87.02、VLA模式83.90 | 该模型下Joint有收益；两种推理路径也不是Fast-WAM配方 |
| world loss之外还有何因素？ | WorldVLA原mask+chunk54.0，改mask76.6，再加world78.1 | 总收益不能全部归到“物理理解” |
| 显式规划何时有证据？ | Cosmos在额外rollout后best-of-8使两困难任务完成分加12.5 | 增加数据、value训练和计算，不是免费提升 |
| 异步更快会不会忽略反馈？ | LingBot同步92.9、反馈校正异步90.4、naive异步74.3 | Easy配置；校正后仍有性能代价 |

最稳妥的综合结论是：**未来监督往往帮助策略学习，但是否值得在线计算未来、是否需要搜索，要看结构、训练配方和任务。** Fast-WAM提供很好的拆分方式；Seer、Motus、Cosmos回答的是不同层次的问题。现有这些表格不足以支持“world model必然学到可迁移三维物理”这一更强主张。

## 5. 评测：数字应该连同问题一起记

| 论文常见数字 | 真实口径 | 误读风险 |
|---|---|---|
| UniPi 77.1% | 生成视频末帧的分类器成功分数 | 写成真机机器人SR |
| GR-1 94.9% / 4.21 | CALVIN链第一项成功率 / 平均连续完成数；完整五项73.1% | 当作完整长任务成功率 |
| Seer 4.28 | CALVIN ABC→D，最佳三个checkpoint平均 | 与当前仓库4.30混写，或看成百分比 |
| Cosmos ALOHA 93.6 | 四任务平均completion score | 当作所有步骤完整成功概率 |
| Motus实机63.22/59.30 | 两平台部分完成分 | 与LIBERO SR直接排榜 |
| DreamZero未见技能39.5 | task progress | 写成零样本完整任务成功率 |
| Action Images 20.6→36.7 | 开环域内任务，增加学习式动作头 | 当作闭环重规划或几何一致性单因素收益 |
| V-JEPA 2-AC杯子pick-place80% | 两实验室平均，图像目标+中间子目标；每单元10trials | 当作自然语言多任务通用策略结果 |

每篇至少记录：训练/测试划分、是否目标域微调、seen/unseen指令、控制器/动作坐标、相机输入、episode与seed、是否选best checkpoint、失败重试、成功/部分进度定义。还要区分**生成质量**（FVD/PSNR）、**离线动作误差**（MSE）、**闭环成功率**、**几何一致性**。没有哪一个标量能替代其余三类。

### 推理速度也有多个含义

Fast-WAM的190ms是单5090D V2的32动作块；GE的200ms是单4090的54动作块；VideoVLA的1.1s是H100短视频+10步去噪；DreamZero-Flash150ms是2×GB200，且一去噪步的任务进度低于四步版；Cosmos best-of-8规划4.9s是8×H100；V-JEPA 2-AC每动作16s还包含800候选搜索。它们不能组成同硬件延迟排行榜。应分别记 **端到端重规划延迟、动作执行频率、chunk长度、执行前缀、吞吐量、GPU与去噪/搜索预算**。

## 6. 跨视角：从主流 WAM 中看到了什么

### 机制层面的四档区别

| 证据类型 | 本库例子 | 已有证据与缺口 |
|---|---|---|
| 多相机输入/拼接 | Fast-WAM、DreamZero、LingBot、Cosmos | 能接收多视角；并无自动的几何或动作不变性保证 |
| 跨视图特征通信 | GE-Base | 指定层跨视图attention；缺严格相机外推/几何损失的控制因果证据 |
| 相机几何进入模型或解码 | Action Images、GE-Sim、PAIWorld | 可建立投影关系；依赖标定/估计，且控制/仿真/视频任务各不相同 |
| 同状态动作/状态一致性与视角测试 | SCVC；鲁棒性研究；V-JEPA相机诊断 | 最直接触及本项目问题，但各自范围与实验限制不同 |

### 一张相机分项表足以改变阅读重点

[鲁棒性研究v5](related-work/wam-robustness.md)在RoboTwin2.0-Plus报告：

| 方法 | 原设置SR | 相机扰动SR |
|---|---:|---:|
| π0.5 | 78.4 | **45.6** |
| Motus | 87.0 | 21.6 |
| LingBot-VA | **92.1** | 28.9 |
| Fast-WAM | 91.2 | 30.4 |

这里默认只改变头部相机距离和小角度朝向，球面位置C2关闭，腕相机保持原样。即便在此范围，相机分项排序也和原设置不同。该表支持“需要专门检验相机变化”，不支持把所有WAM宣布为更差或把差距归因于唯一架构因素。

**已有可学习的机制线索**：V-JEPA发现未输入相机条件时混合左右相机训练会下降；Action Images让同一动作在不同视角呈不同热图、解码回同一物理动作；SCVC限制只对适合不变的量施加一致性。三者共同提示先明确坐标、可见性和信息依赖，再谈约束。这里是阅读综合，尚未成为经本项目实验支持的新方法。

## 7. 怎样按 research-workflow 继续推进

本轮完成的是“抓文献→方法/实验深读→对照与复现准备”。下一步应先复核一个公开baseline的原始结果，再提出研究主张。

| 学习或复现目的 | 优先入口 | 开跑前需要确认 |
|---|---|---|
| 理解辅助预测与未来表征差别 | GR-1→Seer→UVA | GR-1训练链不完整；Seer有较清晰训练入口 |
| 隔离视频监督和部署生成 | Fast-WAM | 固定论文v2模式、数据、attention mask，区分仓库Optional-IDM |
| 学习视频模型的直接适配 | Cosmos Policy | EDM而非flow；completion score定义；规划额外rollout |
| 学习历史与异步执行 | LingBot-VA | 公开shared-backbone与论文dual-stream差异，VA2分开 |
| 研究多模态/潜动作联合训练 | Motus | 光流潜动作与相机运动混淆、完整数据链、模式实际计算量 |
| 接近多视角生成结构 | GE-Base/GE-Act | V1与v2分开；GE-Act和GE-Sim的相机/动作接口不同 |
| 研究动作几何表达 | Action Images | 正文/附录配置冲突、标定误差、解码、开环边界 |
| 建相机泛化测试 | RoboTwin2.0-Plus+SCVC协议 | 明确启用相机轴、同状态配对、held-out位姿、腕相机与控制坐标 |

完整路线见 [学习计划](reading-plan.md)。本轮没有下载模型权重、机器人数据或执行训练；所有复现缺口保留在逐篇笔记中。
