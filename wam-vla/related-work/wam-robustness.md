# Do World Action Models Generalize Better than VLAs? 深读

核查日期：2026-09-25。固定版本：[2603.22078v5](https://arxiv.org/abs/2603.22078v5)，2026-07-30。[原文 PDF](https://arxiv.org/pdf/2603.22078v5)、[TeX（论文入口）](https://arxiv.org/abs/2603.22078v5)。已读方法、主表、附录扰动协议。它是 **评测研究，不是新 WAM 架构**；以下为作者评测，本项目未复现。

## 1. 为什么值得和主流 WAM 一起读

普通 LIBERO/RoboTwin 高成功率能否迁移到相机、噪声、背景变化，是本项目必须回答的问题。这篇把 Fast-WAM、LingBot-VA、Cosmos Policy、GE-Act、Motus 与 VLA 放在结构化扰动下比较，可以直接检查“视频先验意味着更强视角泛化”是否被证据支持。因发表较近，纳入依据是问题直接性和公开评测，不声称已有长期学术影响力。

其 WAM 分类较窄，强调用视频骨干生成动作，将 Motus 列为 VLA+WM。我们的综述使用较宽的历史路线分类；两者标签不同，不代表一方遗漏了该模型。读者应比较计算图而非纠结称谓。

## 2. 评测设计

RoboTwin 2.0-Plus 使用双臂 Aloha-Agilex、头部与双腕三相机，320×240；LIBERO-Plus 是单臂 Franka、外部与腕部两相机，256²。七个变化维度：相机、机器人初始状态、语言、光照、背景、传感器噪声、物体布局。

附录 A 的 RoboTwin 协议是50任务，每任务8个配置（clean+7个单维度分支），每配置50 episodes；完整一模型为 **20,000 episodes（按协议计算）**。每分支只启用一个大维度，维度内可组合子项；不是所有变化同时发生。

### 相机分支的真实范围

- 只改变头部相机，腕相机保持原样。
- C1：距离倍率0.85–1.0。
- C3：yaw/pitch/roll各0–5°，随机正负。
- C2：方位/仰角±10°和距离±10%的球面位置变化，**默认关闭**，因仿真稳定性问题只作为可选消融。

因此主表 Camera 不是完整的视角外推测试，更不是“所有相机都任意移动”。若目标是大角度、未见相机布局泛化，必须另加协议。

## 3. 用什么模型与训练数据

多数方法用官方仓库/checkpoint。RoboTwin 的 π0.5 由作者用27.5k clean+randomized轨迹、60k步、batch64做全量微调；采用JAX实现、delta joint动作。Fast-WAM v5已纳入：RoboTwin使用 `robotwin_uncond_3cam_384`、27.5k多样化数据；LIBERO checkpoint仅用clean演示。**旧版本“尚未评测Fast-WAM”的描述已不适用。**

LIBERO主表混合本研究运行结果和其他论文已报告结果；caption逐一标明，不能视为统一代码/训练预算的大规模重训。DreamZero未进入定量表，作者理由涉及checkpoint与基准不匹配、重训/推理代价；这是本研究选择，不代表DreamZero当前没有公开代码。GigaWorld-Policy也未列入定量，不能据此补一个排名。

## 4. 与相机相关的结果，比总体结论更重要

### RoboTwin 2.0-Plus，Table 3，成功率 %

| 模型 | Original | Camera | Light | Noise | 作者 Total |
|---|---:|---:|---:|---:|---:|
| π0.5 | 78.4 | **45.6** | 49.6 | 64.9 | 58.6 |
| Motus | 87.0 | 21.6 | 84.6 | 43.1 | 71.5 |
| LingBot-VA | 92.1 | 28.9 | **89.0** | **80.9** | **74.2** |
| Fast-WAM | 91.2 | 30.4 | 88.8 | 76.4 | 72.7 |

原表 Total 数值与 **含 Original 在内8列的等权平均**相符（按显示值核算），不是仅七个扰动分支平均；正文的宽泛描述不应覆盖表内算术口径。WAM 总体较好，Camera 上却低于 π0.5；Motus 对机器人初始化85.0%较强，但噪声43.1%并不强。不存在各维度一致胜出的赢家。

### LIBERO-Plus，Table 4，成功率 %

| 模型 | Original | Camera | Robot | Light | 作者 Total |
|---|---:|---:|---:|---:|---:|
| π0.5 | 96.9 | 75.4 | 77.5 | 96.9 | **85.7** |
| Cosmos Policy | 98.5 | **75.8** | 63.3 | 96.5 | 82.2 |
| GE-Act | 94.4 | 60.7 | 77.0 | 95.8 | 80.3 |
| Fast-WAM | 97.6 | 16.4 | 44.5 | 78.2 | 51.5 |

LIBERO 的 Total 也不能从这里七列简单平均推出；原文未在主表caption明确展开聚合权重，因此保留为“作者Total”，不与RoboTwin的Total跨基准相减或拼榜。标准基准97.6并不能保证换相机仍有效。

## 5. 因果结论应该收敛到哪里

作者用Fast-WAM在LIBERO与RoboTwin的差异强调任务数据多样性。这是有价值线索，但两者同时改变了任务、机器人、观察、动作空间和数据量，**不足以单独隔离 domain randomization 的因果贡献**。也不足以断言“joint denoising天生比IDM更依赖数据多样性”。需要在同一任务/骨干/预算下改变数据或依赖方向，才有更强证据。

还有一处架构归类需对照原方法：本研究把Fast-WAM称作jointly denoises state and action，但Fast-WAM原版mask中，动作只读当前条件与动作token，未来视频只读当前条件与视频token，二者不能互读。它同时训练两个目标，却不是VideoVLA式双向视频—动作联合去噪。因而基于这一粗分类解释泛化差异尤其需要谨慎。表3的RoboTwin Original按作者caption称Easy，保留本研究重新评测值，不拿各方法原论文的clean/random数字替换。

π0 的不同实现也说明 checkpoint/接口的重要性：LIBERO Camera从引用旧结果13.8到作者JAX rerun61.0，Total53.6到69.4。不能把这些差异全部归因于架构；应先核输入裁剪、控制接口、权重和评测环境。

## 6. 延迟表的脚注与局限

Table 5：π0.5 63ms/50动作、GE-Act300ms/36、Cosmos390ms/16、LingBot实机配置480ms/32、Motus1175ms/16、LingBot RoboTwin配置5230ms/32。LingBot两种配置分别用3/5和25/50步state/action去噪，不能只报一个不带配置的速度。

表中Fast-WAM190ms/32动作 **脚注明确取自原论文硬件，没有在本研究设备重测**；caption“同一设备”不涵盖这一例外。正文“WAM至少慢4.8倍”也不适用于更新加入的Fast-WAM约3倍这一行。表内未清楚标出通用计时设备型号；不将其作为严格硬件归一化排名。动作块长短不同，推理延迟也不能直接除成相同意义的控制频率。

## 7. 开源与本项目用途

[官方项目](https://robot-robustness.github.io/RoboTwin2.0-Plus/)与[代码](https://github.com/Robot-Robustness/RoboTwin2.0-Plus)均可访问，含环境、扰动配置、政策接口与数据收集入口；没有运行安装或验证全部checkpoint适配。原文称21子维度，网页不同段落又写20加clean；按列出的子项求和是20。使用时以具体配置开关为准，不能只记录宣传数字。

适合作为未来复现的诊断协议来源：先复核clean→分别复核相机/噪声→再扩到全维度。还需补跨视角课题真正关心的同状态配对动作差异、held-out角度、腕部视角变化，以及一致性改善是否能提高闭环成功率。本文展示相机短板，但没有实现跨视角一致性方法，也没有验证 SCVC 或 Action Images。
