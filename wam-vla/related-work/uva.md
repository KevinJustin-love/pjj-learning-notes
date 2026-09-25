# UVA 阅读笔记

核实日期：2026-09-25。论文：[Unified Video Action Model](https://arxiv.org/abs/2503.00200)，Shuang Li、Yihuai Gao、Dorsa Sadigh、Shuran Song，arXiv:2503.00200；初稿 2025-02-28，当前 v3 为 2025-04-24；[官网](https://unified-video-action-model.github.io/)标注 RSS 2025。以下基于 [v3 正文](https://arxiv.org/html/2503.00200v3)和[同版本PDF](https://arxiv.org/pdf/2503.00200v3)，均为作者报告，未复现。UVA 不是名称相似的 UniVLA。**HTML转版的表号与PDF不一致，以下优先注明PDF表号与页序（从1开始）。**

## 为什么值得读

UVA 自称统一视频动作模型，目标是让同一表示支持策略、视频生成和正/逆动力学。Fast-WAM §2明确引用它，故“训练视频、推理只出动作”不能单独作为新的研究贡献。值得精读的是：训练时如何让两种监督共享表示，推理时又如何只付所需解码成本。下文先给整体结论，再展开信息流和实验边界。

## 结构与训练

UVA 先学一个共用的视频—动作潜表示，再让两个小 diffusion head 分别解码视频和动作。输入历史图像与历史动作块，可加语言；图像通过预训练 kl-f16 VAE，动作通过线性投影，并与未来掩码视觉 token 融合。骨干继承 MAR-B 图像生成模型，论文报告整体约 0.5B；LIBERO10 语言用 CLIP text encoder。视频帧率可低于动作频率，一帧对应多步动作。证据：§III-A/B、图2、§V-D、附录X-A。

训练同时最小化视频和动作的噪声预测 MSE，`L=L_action+L_video`；掩码允许切换 policy、video model、forward dynamics、inverse dynamics 和 policy+planner。官网及代码推荐先训练视频，再联合训练视频和动作。作为 policy 部署时，Transformer 生成共用潜表示，**跳过视频 diffusion/VAE 解码，只运行动作头**；作为规划器才生成视频评价候选动作。因此“训练视频、测试不出视频”已有明确先例；Fast-WAM 自身 §2 也引用 UVA。证据：§III-C/D，官网 Q&A。

## 数据与实验

仿真覆盖 PushT、Toolhang、PushT-M、LIBERO10；真机从公开 UMI cup/towel/mouse 数据各选 500 条，三任务模型共 1500 条，在 ARX X5 上评测，每任务 20 次。LIBERO10 主设置只用第三人称图像，没有腕部相机或本体状态；因此不能把它和用更多视觉/状态输入的四套 LIBERO 均值当作同协议排名。证据：§V、§V-D、附录X-C。

| 作者报告 | UVA | 对照 | 证据 |
|---|---:|---:|---|
| LIBERO10，500 次测试 | 0.90 | UVA-action 0.86；π0 0.85 | PDF表I，第5页；HTML表II |
| PushT-M 指标 | 0.88 | DP-C 0.68；UVA-action 0.46 | PDF表I；§V-A定义为平均reward |
| 真机三任务成功率 cup/towel/mouse | .65/.70/.80 | DP-UMI .50/.70/.40 | PDF表II，第6页；HTML表III |
| 真机单任务 cup 成功率 | .85 | DP-UMI .95 | PDF表II |

UVA并非所有设置都优于其他方法：单任务 cup、Toolhang 都存在强于 UVA 的基线；跨方法差异不能单独归因于视频辅助目标。补充实验加入 3175 条无动作人类视频，LIBERO10 的 500-test 指标仅从 0.90 到 0.91；30-test 则为 0.93 到 0.97。必须写明版本和测试次数，不能混用 0.93 与 0.90。附录X-B的一句“每任务10 runs、总500”有文字数量不一致；主文§V-A和PDF表VIII明确500测试对应每任务50次，复现时以脚本确认。证据：附录X-E、PDF表VIII第15页（HTML表VII）。

推理计时也分设置：仿真 L40 上一个动作轨迹为 0.23s；真机 RTX 3080 上 16步动作、16步动作去噪为 95ms，Flash Attention 为 85ms。附录X-D给出编码、Transformer、动作头分解；这些数不能直接和 Fast-WAM 不同 GPU/分辨率/动作长度的 190ms 比快慢。视频前向规划另有 BlockPush 实验：DP 成功率 38%，加 UVA 选动作轨迹为 60%，同一候选流程改用真实模拟器评分为75%（§VII、官网），说明测试时预测未来在显式候选选择中有另一种用途。

## 开源与跨视角判断

[官方代码](https://github.com/ShuangLI59/unified_video_action)有 PushT Colab、PushT/PushT-M/LIBERO10/UMI 权重及训练评测脚本，适合作为较易开始的仿真复现。作者推荐至少4 GPU训练，官网 UMI 示例为8 H100、视频阶段2天+联合阶段2天；不能把可单卡部署等同于便宜重训。仓库基础训练说明没有额外视频数据，附录人类数据扩展应单独匹配配置。

本篇重点是统一表示和解码分离；LIBERO 设置使用单第三人称视角，未设计跨相机一致性损失或评测。它支持研究“视频监督是否有助策略”，不能直接作为“跨视角一致性已解决”的证据。

## 深读机制：按 shape 追踪一次训练与推理

### 输入如何对齐

§III 设历史图像 `O[t−h+1:t]`、历史动作块 `A[t−h:t−1]`，预测 `h'` 个未来图像及动作块；实验设 `h=h'`。每块 `A_t∈R^(L×m)`，`L` 为动作数、`m` 为动作维度，不能把一个图像时刻理解成只输出一步控制。

图像经 kl-f16 VAE 成为连续 latent map，再展平为每帧 `N` 个、维度 `d` 的视觉 token。历史动作比图像采样更密：一个图像时刻对应一个动作块，动作块重复、线性投影后与 `N` 个视觉位置对齐。未来训练图像也经过 VAE，随机遮住部分 latent 位置。

**这里不是把所有模态直接接成长 token 序列。** §III-B 先把同一位置的历史图像、历史动作与遮蔽后的未来视觉表示沿 **channel 维拼接**，再沿时间形成 `N×h` 个位置。Transformer 输出联合 latent `Z∈R^(h×N×d)`。语言任务使用重复的 CLIP 文本表示参与融合；论文称其为 cross-attention，同时描述把文本 token 追加到序列，复现应结合代码配置，不能仅由名称假定某个固定 cross-attention 模块。

未来帧使用**相同空间位置的 mask**，防止从另一未来帧同一位置抄答案。生成时从全 mask 开始，每轮同时生成所有帧的同一组空间位置，再逐轮加入已经生成的内容。这里的 autoregressive 是逐组补全 latent 位置，不能误写成完整视频按帧严格左到右生成。增加补全轮数与增加 diffusion 去噪步数也是两件事（§III-B、附录X-A）。

### 输出与梯度流向

视频头对每个 `z_i` 生成对应的连续视觉 latent patch，再由 VAE decoder 合成图像；动作头先用卷积与 MLP 汇聚一帧的 `N` 个 latent，条件生成完整 `L×m` 动作块。两者都是轻量 diffusion decoder，不将完整 Transformer 在每次去噪中重跑（§III-C）。

训练的两个目标为：

`Laction = E ||ε−εθ(Ak,k,Z)||²`

`Lvideo = E (1/N) Σi ||εi−εφ(Oik,k,zi)||²`。

两者相加再对时间求和。视频梯度促使共享 `Z` 保留场景变化信息；动作梯度使其服务控制。它们作用于连续动作/连续视觉 latent，没有离散 action vocabulary，也不是 flow matching。使用 MAR/VAE 预训练权重，因此“没有额外机器人数据”不能扩写成从零训练。

§III-D 用可学习 mask token 替换不需要的输入，并按任务选择输出损失：观察到动作是策略，观察到未来观察是视频模型；观察加候选动作预测未来是前向动力学；前后图像推动作是逆动力学。改变输入可见性才使一个模型承担多种角色；无动作视频可以参与视频损失，但不会凭空产生动作标签。

策略推理时把未知未来设 mask，只运行共享 latent 计算和动作头。若选择视频生成或候选评分，才调用视频头。因此 UVA 与 UniPi 的“显式视频先生成、独立逆动力学后翻译”不同；也不能直接等同 Fast-WAM 的 Wan DiT/MoT及禁止动作读取未来视频的 attention mask。

## 深读实验：对照到底隔离了什么

**UVA-action消融。** PDF表I，PushT的 `.98 vs .45`、Toolhang的 `.88 vs .62`，加上前述多任务结果，支持联合训练有助这些设置中的策略。它同时移除了视频生成部分，并非控制所有参数与训练 FLOPs 后的纯损失单因素试验；不能仅靠它证明提升唯一来自三维几何知识。表I 中 DP-C 的 Toolhang为 `.95`，仍优于UVA。仿真主结果还报告最佳 checkpoint 的成绩，复现需要匹配 checkpoint 选择协议（§V-A/D）。

**视频质量。** §VI、PDF表IV在500个生成视频上计算FVD：LIBERO10一轮补全89.36、八轮51.10，Cup为51.34→29.72；UniPi重实现为56.55/71.37。因此一轮UVA在LIBERO10并不优于该对照。较低FVD不是物理可执行性或多相机一致性的证明；补全轮数的增益也不意味着直接策略必须使用八轮视频生成。

**前向规划。** §VII、PDF表V的BlockPush，每步DP-C采100条候选、每条16动作，UVA预测未来视频，按方块到指定目标的距离评分，执行选中轨迹前6步后再采样；四个指定配对各测试10次。38%→60%的改善包含候选搜索、奖励计算及闭环重规划。真实模拟器的75%是该候选/检测流程的对照成绩，受候选质量和物体检测限制，**不是任务理论上界**。

**逆动力学。** §VIII、PDF表VI在未见UMI Cup数据上，以Mocap相机位姿为真值：UVA平移/旋转L2误差为 `0.75cm/1.11°`，UniPi逆动力学重实现为 `1.92cm/2.21°`，视觉惯性SLAM为 `0.41cm/0.30°`。这是位姿误差，不能与任务成功率混用；UVA仍未达到SLAM精度。

**mask比例。** 附录X-F、PDF表IX比较application-dependent和application-independent策略；后者50% mask的策略分数.87，而25%对视频/前向动力学FVD更好。不同目标需要不同取舍，不能默认遮得越多越鲁棒。这也不是跨相机遮挡消融。

## 复现前需要对齐的具体条件

1. 仿真表I通常预测16动作、执行8动作；OpenVLA为单步模型，需调用8次匹配执行数。UVA仿真动作头使用100去噪步，真机使用16步。不能只比较表格中的毫秒而忽略调用次数（§V-D）。
2. PDF表VII、附录X-D将RTX3080延迟分为VAE编码40ms、Transformer40ms、动作头15ms；100动作去噪步时动作头93ms，总173ms。视频头若调用，16/100去噪步还需100/625ms。省略视频生成后仍有图像编码和动作采样成本。
3. [官方README](https://github.com/ShuangLI59/unified_video_action)列出LIBERO重放后转换的绝对动作、UMI抽样索引、`different_history_freq`和镜面区域遮蔽。匹配这些数据/控制频率约定比只匹配网络参数更重要。仓库的代码许可为MIT，数据需按原来源许可使用；权重链接存在不代表本次下载验证过。
4. 官方默认并行展开UMI数据可能需要很大主存，README给出串行处理替代；不要直接把大型训练命令视为本机可执行预算。本次没有安装、下载权重或运行实验。

## 对本项目的可检验启发与证据入口

相同位置跨帧mask处理的是时间信息泄漏，不是不同相机的对应关系。若加入跨视角约束，应在相同输入、动作坐标、数据与算力下保留UVA-action以及普通双视角视频辅助目标的对照，分别测移动相机、遮挡、缺失相机、任务成功率和状态一致性。这是研究建议，不是本文已经完成的结论。

已用 [paper.txt（论文入口）](https://arxiv.org/abs/2503.00200v3)、`source/main.tex`实际引入的 [method.tex（论文入口）](https://arxiv.org/abs/2503.00200v3)、[experiments.tex（论文入口）](https://arxiv.org/abs/2503.00200v3)、[supplementary.tex（论文入口）](https://arxiv.org/abs/2503.00200v3)，及 `source/tables/` 中主结果、视频、MPC、逆动力学、延迟、human_data、maskfilter表复核。未使用未完成的 `history_len.tex` 空表或 `.orig` 文件作为结果。
