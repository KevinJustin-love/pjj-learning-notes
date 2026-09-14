# AR Video Gen

:date: 08/19/2026

:writing_hand: Kevin Justin

## Glossary

- **AR**：Autoregressive。
- **Causal**：当前输出只能使用部署时已经可用的过去信息 / control。
- **Bi-directional**：模型可以同时访问当前 sequence 的 past / future context。
- **VAE**：Variational Autoencoder，video diffusion 常在 continuous latent space 工作。
- **VQ**：Vector Quantization，把 continuous latent 映射成 discrete codebook IDs。
- **BSQ**：Binary Spherical Quantization，用 bitwise discrete representation 表示 visual features。
- **DiT**：Diffusion Transformer。
- **PF-ODE**：Probability Flow ODE，用 deterministic ODE 表达 diffusion probability flow。
- **NFE**：Number of Function Evaluations，常用于衡量 diffusion sampling steps / network evaluations。
- **TF**：Teacher Forcing。
- **SF**：Self-Forcing。
- **CM**：Consistency Model。
- **DMD**：Distribution Matching Distillation。
- **KV Cache**：缓存 previous causal attention 的 key/value，减少 streaming generation 重算。
- **Exposure Bias**：训练使用 ideal / GT history，推理使用 model-generated history 导致的 distribution gap。
- **Mode Covering**：倾向覆盖 target distribution 更多 modes。
- **Mode Seeking**：倾向集中在 target distribution 的 high-density modes。
- **Next-scale Prediction**：一次预测一个 coarse-to-fine scale token block。
- **Residual Token Block**：在当前 scale 上补充 previous scales 尚未表达的信息的一整块 discrete tokens。

## Overall Paradigm

从「AR video model 是怎么得到的」这个角度看，目前主要有两条路线：

1. **Distill / convert into AR**：从一个很强的 bi-directional video diffusion model 出发，先 causalize，再做 few-step distillation 和 self-forcing refinement。
2. **Native AR training**：训练阶段直接使用 causal / autoregressive objective。这里又可以分为：
   - **continuous AR diffusion / flow**：代表 Diffusion Forcing、MAGI-1；
   - **discrete next-scale AR**：代表 VAR -> Infinity -> InfinityStar。

因此，一个更清楚的 taxonomy 是：

```text
AR Video Generation
│
├── Route 1: Distill / convert into AR
│   ├── CausVid
│   ├── Self-Forcing
│   ├── Causal Forcing / Causal Forcing++
│   ├── Causal-rCM
│   └── CMD
│
└── Route 2: Native AR
    ├── Continuous AR diffusion / flow
    │   ├── Diffusion Forcing
    │   └── MAGI-1
    │
    └── Discrete next-scale AR
        ├── VAR
        ├── Infinity
        └── InfinityStar
```

这里需要注意，AR video model 常见的数学形式有两类。

### 1. Temporal AR + conditional diffusion / flow

把视频分为 frame/chunk：

$$
p(x_{1:C}\mid y)
=
\prod_{c=1}^{C}
p_\theta(x_c\mid x_{<c},y).
$$

这里：

- $c$：temporal frame / chunk index；
- $x_c$：当前 frame/chunk 的 continuous latent；
- $x_{<c}$：已经生成的历史；
- 每一个 $p_\theta(x_c\mid x_{<c},y)$ 由 conditional diffusion / flow model 参数化。

所以它有两层 generation process：

```text
video time:
chunk 1  ->  chunk 2  ->  chunk 3  ->  ...
   │           │           │
   ↓           ↓           ↓
denoise      denoise     denoise
```

外层是 temporal AR，内层是 diffusion / flow sampling。

代表：CausVid、Self-Forcing、Causal-rCM、MAGI-1。

### 2. Temporal clip AR + spatial next-scale AR

InfinityStar 使用 discrete spacetime pyramid。概念上可以写成：

$$
p(r\mid y)
=
\prod_c\prod_k
p_\theta
\left(
  r_{c,k}
  \mid
  \mathcal H_{c,k}, y
\right),
$$

其中：

- $c$：**temporal clip index**，表示视频时间轴上的第几段；
- $k$：**spatial scale index**，表示当前 clip 做到第几个 coarse-to-fine level；
- $r_{c,k}$：第 $c$ 个 clip、第 $k$ 个 scale 上的 **discrete residual token block**；
- $\mathcal H_{c,k}$：当前 step 允许访问的历史 token。

如果按完整的逻辑生成顺序理解，可以把 history 写成：

$$
\mathcal H_{c,k}
=
\{r_{c',k'}: c'<c\}
\cup
\{r_{c,k'}: k'<k\}.
$$

也就是：previous clips + current clip previous scales。

InfinityStar 的实际 Spacetime Sparse Attention 会进一步压缩 temporal history，主要保留前一个 clip 的 last scale，并保留当前 clip 已生成的 scales：

$$
p_\theta(r_{c,k}\mid r_{c,<k},r_{c-1,K},y).
$$

> 注：这里的 $k$ 才是 scale；$r_{c,k}$ 是这个 scale 上真正需要预测的数据。

---

## Route 1：从 bi-directional teacher 蒸馏得到 causal AR

这一条路线的基本 motivation 很直接：

- 大规模 bi-directional video diffusion 已经有很强的 visual prior；
- full-sequence bidirectional attention 很难支持低延迟 streaming；
- multi-step diffusion 的 NFE (Number of Function Evaluations，生成一次需要神经网络的 forward 次数) 很高；
- 目标是复用原模型能力，同时得到 causal + few-step 的 generator。

一个典型 pipeline 可以写成：

$$
\text{bidirectional teacher}
\rightarrow
\text{causal teacher / causal initialization}
\rightarrow
\text{few-step causal student}
\rightarrow
\text{on-policy refinement}.
$$

> On-policy refinement 指的是让 student 在自己真实会生成出来的轨迹上继续训练，而不是只在 ground-truth / teacher 提供的轨迹上训练。如 Self-forcing，DMD

### 方法演化

| 方法 | 核心 recipe | 主要修复什么 |
| --- | --- | --- |
| **CausVid** | bidir teacher -> ODE initialization -> causal 4-step student -> asymmetric DMD | bidirectional -> causal；50-step -> few-step |
| **Self-Forcing** | student 自己 rollout history，再做 video-level DMD | exposure bias |
| **Causal Forcing** | 先得到 causal diffusion teacher，再从 causal teacher 做 ODE distillation | bidir teacher -> causal student 的 flow / information mismatch |
| **Causal Forcing++** | causal consistency distillation 替代完整 PF-ODE trajectory | frame-wise 1–2 step initialization cost |
| **Causal-rCM** | $\boxed{\text{TF}\rightarrow\text{TF-CM}\rightarrow\text{SF-DMD}}$ | mode coverage + on-policy quality |
| **CMD** | causal teacher + causal DMD scoring + Prefix Scoring / Prefix Corruption | DMD teacher 偷看未来的问题 |

### CausVid

CausVid 是这一类方法很重要的早期工作。

主要贡献：

1. 把 pretrained bidirectional video diffusion transformer 改成 causal autoregressive transformer；
2. 用 video DMD 把原本几十步的 diffusion sampling 压缩到 few-step。

核心 recipe：

$$
\text{bidir teacher}
\rightarrow
\text{ODE trajectory initialization}
\rightarrow
\text{causal student}
\rightarrow
\text{asymmetric DMD}.
$$

### Self-Forcing：解决 exposure bias

Exposure bias 指训练时 context 通常来自 ground-truth，而 inference 时 context 来自 student 自己之前生成的结果。

Self-Forcing 在训练阶段直接做 autoregressive rollout：

$$
\hat x_1
\rightarrow
\hat x_2
\rightarrow
\hat x_3
\rightarrow\cdots
$$

后续 frame/chunk condition 在 student-generated history 上，并把整段生成视频当成一个整体来优化。

Self-Forcing 还结合 KV cache rollout 和 stochastic gradient truncation，让长 rollout training 的显存和计算更可控。

### Causal Forcing：teacher 的 information boundary 也需要匹配

Self-Forcing 主要处理 generated-history mismatch。Causal Forcing 进一步分析了另一个问题：**bidirectional teacher 和 causal student 的 conditional flow 并不天然对齐。**

当不同 future 对应不同 teacher target 时，causal student 看到相同 causal input，却可能收到不同的 regression target。

因此它先训练 / 构造一个 many-step causal diffusion teacher，再从 causal teacher 做 few-step ODE distillation：

$$
\text{bidir base}
\rightarrow
\text{causal diffusion teacher}
\rightarrow
\text{few-step causal student}.
$$

这样 teacher 和 student 的 causal information set 更一致。

### Causal Forcing++：把 causal initialization 做到 1–2 step

完整 PF-ODE trajectory distillation 很贵。Causal Forcing++ 用 causal consistency distillation，利用相邻 noise timestep 的一次 online teacher ODE step 提供 supervision。

概念上：

$$
x_t
\xrightarrow{\text{teacher one ODE step}}
x_s,
$$

然后约束 consistency student：

$$
f_\theta(x_t,t)
\approx
f_{\bar\theta}(x_s,s).
$$

这样可以学习 causal conditional flow map，同时降低预计算和存储完整 ODE trajectory 的成本。

### Causal-rCM：TF -> TF-CM -> SF-DMD

Causal-rCM 的核心思想是把两类 objective 组合起来：

- **Teacher-Forcing / Consistency Matching**：offline、稳定、coverage 较好；
- **Self-Forcing / DMD**：on-policy、直接优化 inference rollout，偏向 high-density modes。

完整 recipe：

$$
\boxed{
\text{TF}
\rightarrow
\text{TF-CM}
\rightarrow
\text{SF-DMD}
}
$$

可以记成：

```text
TF
先学会 causal AR diffusion
   ↓
TF-CM
把 many-step causal diffusion 压成 few-step
   ↓
SF-DMD
在 student 自己的 rollout 上继续做 distribution matching
```

### CMD

CMD = Context-Matched Distillation，进一步要求 teacher score 的 information set 和 student generation 时保持一致：

$$
s_{T,t}
=
s_T(\hat x_t^\tau\mid \hat x_{<t},c_{\le t},\tau).
$$

它还加入 **Prefix Scoring**：teacher 评价当前 target 时，condition 在真正产生这个 target 的 cached student-generated prefix 上。

另外使用 **Prefix Corruption** 提升 teacher 对 early-stage imperfect prefix 的鲁棒性。

---

## Route 2A：Native continuous AR diffusion / flow

这一类方法在 pretraining 阶段直接建立 temporal causality。

其目标依然可以写成：

$$
p(x_{1:C}\mid y)
=
\prod_{c=1}^{C}
p_\theta(x_c\mid x_{<c},y),
$$

其中 conditional distribution 使用 diffusion / flow 建模。

### Diffusion Forcing

Diffusion Forcing 的核心点可以记成：

> 每一个 temporal token 都有自己的 diffusion clock。

传统 full-sequence diffusion 常让整个 sequence 使用同一个 noise level：

$$
(x_1^k,x_2^k,\ldots,x_T^k).
$$

Diffusion Forcing 允许：

$$
(x_1^{k_1},x_2^{k_2},\ldots,x_T^{k_T}),
$$

不同时间位置可以处于不同 noise level。

训练时对 per-token noise levels 做随机化，可以覆盖很多不同的 causal prediction / denoising setting。

这个视角把两个轴放到同一个框架：

- horizontal axis：sequence / video time；
- vertical axis：diffusion time。

### MAGI-1

MAGI-1 可以看作大规模 chunk-wise native AR diffusion。

视频被切成固定长度 chunks：

$$
C_1,C_2,\ldots,C_N.
$$

然后：

$$
p(C_{1:N}\mid y)
=
\prod_i p_\theta(C_i\mid C_{<i},y).
$$

每个 $p(C_i\mid C_{<i},y)$ 由 flow / diffusion model 表示。

#### Progressive noise

MAGI-1 让 noise level 沿 temporal chunk 单调增大：

$$
t_i<t_j,\quad i<j.
$$

所以训练输入形成：

```text
past                                  future
more clean -> less clean -> noisy -> more noisy
```

#### Attention structure

MAGI-1 的 attention 可以记成：

- **within chunk**：full attention；
- **across chunks**：causal attention。

因此一个 chunk 内可以联合建模短时间 motion，chunk 之间保持 streaming causality。

#### Pipeline denoising

逻辑生成顺序依然是：

$$
C_1\rightarrow C_2\rightarrow C_3.
$$

计算上，前一个 chunk denoise 到一定程度之后，后一个 chunk 可以提前启动，形成 overlapping / pipelined denoising。

#### Few-step acceleration

MAGI-1 的 base model 来自 native causal training。为了进一步减少 NFE，它还使用 Shortcut Model 做 sampling acceleration。

所以 MAGI-1 的路线可以概括为：

$$
\text{native causal diffusion pretraining}
\rightarrow
\text{sampling acceleration / distillation}.
$$

### Native continuous AR 和 distilled AR 的区别

| 维度 | Distilled AR | Native continuous AR |
| --- | --- | --- |
| causality 从哪里来 | post-training / distillation 阶段加入 | pretraining 阶段直接建立 |
| 是否复用强 bidirectional teacher | usually yes | 可以不依赖 |
| conditional distribution | diffusion / flow | diffusion / flow |
| few-step 问题 | 核心问题之一 | base model 仍可能 many-step，后续同样需要加速 |
| 优势 | 复用成熟 foundation model | train / inference causality 更统一 |
| 成本 | post-training 成本低于重新大规模 pretrain | 原生 causal pretraining 成本更高 |

---

## Route 2B：Native discrete next-scale AR

这一条路线和 LLM 风格的 discrete autoregression 更接近。

核心演化：

$$
\boxed{
\text{VQ tokenizer}
\rightarrow
\text{VAR}
\rightarrow
\text{Infinity}
\rightarrow
\text{InfinityStar}
}
$$

### VQ tokenizer

VQ = Vector Quantization。

目标：把 continuous visual latent 变成 discrete visual tokens。

一个典型 VQ tokenizer：

$$
x
\xrightarrow{Encoder}
z
\xrightarrow{Vector\ Quantization}q(z)
\xrightarrow{Decoder}\hat x.
$$

假设 codebook：

$$
E=\{e_1,e_2,\ldots,e_K\}.
$$

对于 continuous latent vector $z_{ij}$，选择最近的 code：

$$
q(z_{ij})
=
\arg\min_k\|z_{ij}-e_k\|^2.
$$

最后一个 visual latent position 可以表示为一个整数 token ID：

$$
z_{ij}\rightarrow 137.
$$

这样 Transformer 可以像 language model 一样预测 discrete visual token。

VQ tokenizer 的基本 trade-off：

- codebook 太小：信息容量低；
- codebook 很大：输出 classifier / codebook optimization 成本升高；
- quantization 会引入 reconstruction bottleneck。

### VAR：Visual Autoregressive Modeling

传统 visual AR 常使用 raster-scan next-token prediction：

$$
r_1\rightarrow r_2\rightarrow\cdots\rightarrow r_N.
$$

如果 latent map 很大，sequential AR steps 会很多。

VAR 将 AR unit 改成 **scale**：

$$
1\times1
\rightarrow
2\times2
\rightarrow
4\times4
\rightarrow
8\times8
\rightarrow
16\times16.
$$

factorization：

$$
\boxed{
p(r_1,\ldots,r_K)
=
\prod_{k=1}^{K}
p(r_k\mid r_{<k})
}
$$

称为 **next-scale prediction**。

这里：

- $k$ 是 scale index；
- $r_k$ 是第 $k$ 个 scale 上的一整块 token map；
- 同一 scale 的大量 token 可以并行预测；
- autoregressive dependency 主要发生在 scale 之间。

#### 为什么叫 residual scale / residual token block？

multi-scale tokenizer 采用 coarse-to-fine residual reconstruction。

概念上：

$$
F_0=0,
$$

第 1 个 scale 编码最粗的 feature：

$$
F_1=\operatorname{up}(r_1).
$$

第 2 个 scale 继续编码前一层还没有覆盖的信息：

$$
r_2\approx F-F_1.
$$

累积后：

$$
F_2
=
\operatorname{up}(r_1)
+
\operatorname{up}(r_2).
$$

继续：

$$
F_K
\approx
\sum_{i=1}^{K}\operatorname{up}(r_i)
\approx F.
$$

所以 $r_k$ 表示第 $k$ 层新增的 residual information。

`residual token block` 可以拆成：

- **residual**：补充 previous scales 仍然缺失的信息；
- **token**：经过 discrete quantization；
- **block**：一次预测一整块 token map。

### Infinity：bitwise visual AR

Infinity 继承 VAR 的 next-scale AR，重点修改 visual tokenizer / classifier。

传统 VQ token 是一个 categorical ID：

$$
y\in\{1,2,\ldots,V\}.
$$

如果 vocabulary 极大，输出层需要很大的 $V$-way classifier。

Infinity 将 visual token 表示成 bit vector：

$$
y\leftrightarrow[b_1,b_2,\ldots,b_d].
$$

理论 vocabulary：

$$
V=2^d.
$$

Transformer 直接预测 $d$ 个 binary logits，因此 classifier complexity 从和 $2^d$ 相关的形式，降低到近似和 $d$ 线性相关。

Infinity 使用 Binary Spherical Quantization (BSQ) 构造这类 discrete representation。

#### Bitwise Self-Correction

VAR / next-scale AR 同样存在 train-test mismatch：

训练时 previous scales 通常来自 ground-truth tokenizer：

$$
r_{<k}^{GT},
$$

推理时 previous scales 来自 model prediction：

$$
\hat r_{<k}.
$$

Infinity 使用 Bitwise Self-Correction 模拟 previous-scale prediction errors，并让后面的 scale 学会 correction。

它和 Self-Forcing 有相似 intuition：training context 需要覆盖 inference-time error。

两者的实现空间不同：

- Self-Forcing：continuous video rollout；
- Bitwise Self-Correction：discrete bit / scale prediction error。

### InfinityStar：Spacetime Pyramid Modeling

InfinityStar 将 image next-scale AR 扩展到 video。

最重要的 mental model：

> **temporal clip AR × spatial scale AR**

#### 1. temporal clip $c$

$c$ 表示视频时间轴上的第几段：

```text
video time ->

clip 1        clip 2        clip 3
first frame   next 5s       next 5s
```

第一部分使用 image pyramid 建立 static appearance；后续部分使用 clip pyramids 表示动态视频。

#### 2. spatial scale $k$

每个 clip 内都有 $K$ 个 spatial scales：

$$
(T,h_1,w_1)
\rightarrow
(T,h_2,w_2)
\rightarrow\cdots\rightarrow
(T,h_K,w_K).
$$

关键点：

$$
T\ \text{保持固定},
$$

主要增长的是：

$$
h_k,w_k.
$$

例如：

$$
(4,1,1)
\rightarrow
(4,2,2)
\rightarrow
(4,4,4)
\rightarrow
(4,8,8).
$$

即同一段 temporal clip 在空间上逐步 coarse-to-fine。

早期 scale 已经包含 temporal dimension，因此能够编码很粗的 motion；后期 scale 继续补充空间结构和 local visual details。

#### 3. residual token block $r_{c,k}$

$r_{c,k}$ 表示：

> 第 $c$ 个 temporal clip，在第 $k$ 个 spatial scale 上新增的一整块 discrete residual tokens。

例如：

$$
r_{2,3}\in\mathcal V^{4\times4\times4}.
$$

可以解释成：

- temporal length = 4；
- spatial resolution = $4\times4$；
- 整个 block 有 $4\times4\times4=64$ 个 spatiotemporal token positions。

如果每个 token 又用 $d$ bits：

$$
r_{2,3}\in\{-1,+1\}^{4\times4\times4\times d}.
$$

所以：

$$
(c,k)=\text{address},
$$

$$
r_{c,k}=\text{content at that address}.
$$

#### 4. 生成顺序

可以把 InfinityStar 看成二维 table：

```text
                         spatial scale k ->

                  k=1       k=2       k=3       ...       k=K

clip c=1        r_1,1     r_1,2     r_1,3                r_1,K

clip c=2        r_2,1     r_2,2     r_2,3                r_2,K

clip c=3        r_3,1     r_3,2     r_3,3                r_3,K
   ↓
temporal
```

逻辑顺序：

```text
clip 1: coarse -> medium -> fine
                        ↓
clip 2: coarse -> medium -> fine
                        ↓
clip 3: coarse -> medium -> fine
```

也就是：

$$
r_{1,1}\rightarrow\cdots\rightarrow r_{1,K}
\rightarrow
r_{2,1}\rightarrow\cdots\rightarrow r_{2,K}
\rightarrow\cdots
$$

#### 5. 为什么 history 常写 previous clips + current previous scales？

预测 $r_{c,k}$ 时，完整逻辑历史包含：

$$
r_{<c,*}
$$

以及：

$$
r_{c,<k}.
$$

例如预测 $r_{3,2}$，理论上 previous generated blocks 包含：

$$
r_{1,1:K},\quad r_{2,1:K},\quad r_{3,1}.
$$

InfinityStar 为了控制 long-context attention cost，实际使用 Spacetime Sparse Attention，只保留更稀疏的 temporal context。一个关键设计是当前 clip 主要 attend 前一个 clip 的 last scale。

所以 implementation intuition 更接近：

$$
p_\theta(r_{c,k}\mid r_{c,<k},r_{c-1,K},y).
$$

#### 6. Visual tokenizer

InfinityStar 继承 pretrained continuous video VAE 的 architecture / weights，并插入 parameter-free BSQ quantizer，再进行 image + video fine-tuning。

这样可以：

- 复用 continuous video tokenizer 已经学到的 visual knowledge；
- 降低 discrete video tokenizer from-scratch training difficulty；
- 提升 reconstruction convergence。

#### 7. Stochastic Quantizer Depth (SQD)

spacetime pyramid 中，early scales token 很少，late scales token 很多。tokenizer 容易把大量信息都放到最后几个 scales。

SQD 在 training 时随机丢掉部分 late scales，迫使 early scales 也保存有意义的 global semantic / motion information。

这样会让 next-scale Transformer 更容易学习 coarse-to-fine dependency。

#### 8. Semantic Scale Repetition (SSR)

early scales token 数量少，但决定 global structure 和 coarse motion。

InfinityStar 会对部分 semantic scales 做 repetition / refinement，从很低的额外 token cost 换取更好的结构和 motion dynamics。

#### 9. Spacetime RoPE

为了同时编码：

- scale；
- time；
- height；
- width；

InfinityStar 对 positional encoding 做 spacetime extension，让 Transformer 能区分 token 在整个 spacetime pyramid 中的位置。

---

## Diffusion AR vs Next-scale AR

两类模型都可以 streaming，但内部计算图差异很大。

### Diffusion AR

```text
chunk c
   │
noise
   ↓
denoise step 1
   ↓
denoise step 2
   ↓
clean chunk
   │
   ↓
chunk c+1
```

对应：

$$
p(x_c\mid x_{<c},y).
$$

一个 temporal AR unit 内仍然包含 iterative diffusion / flow sampling。

### InfinityStar

```text
clip c
   │
scale 1 residual block
   ↓
scale 2 residual block
   ↓
scale 3 residual block
   ↓
scale K residual block
   │
   ↓
clip c+1
```

对应：

$$
p(r_{c,k}\mid\text{history},y).
$$

一个 temporal clip 内通过 coarse-to-fine residual token blocks 逐层构建 representation。

可以做一个 engineering-level analogy：

$$
\text{diffusion denoising steps}
\leftrightarrow
\text{next-scale refinement steps}.
$$

两者的数学对象不同：

- diffusion step：持续更新同一个 continuous latent；
- next-scale step：增加一层新的 discrete residual information。

---

## DMD：Distribution Matching Distillation

DMD 在 few-step diffusion distillation 中非常重要。

### Trajectory / regression distillation

一种直接方式是对同一个 noise $z$，让 student 接近 teacher 最终 sample：

$$
G_\theta(z)\approx G_T(z).
$$

这样会绑定 teacher 的 sample-level mapping / sampling trajectory。

### Distribution matching

DMD 直接希望：

$$
p_\theta(x)\approx p_T(x).
$$

student 可以采用和 teacher 很不同的 sampling process，只要最终 distribution 接近 teacher。

DMD 常从 KL gradient 推导出两个 score function 的差：

$$
\nabla_x\log\frac{p_T(x)}{p_\theta(x)}
=
s_T(x)-s_\theta(x).
$$

实际训练里：

- **teacher / real score network**：估计 target distribution score；
- **fake score network**：估计当前 student-generated distribution score；
- 两者的差给 student 提供 distribution correction direction。

简化图：

```text
noise -> student -> fake sample -> add diffusion noise
                           │
                 ┌─────────┴─────────┐
                 ↓                   ↓
           teacher score        fake score
                 │                   │
                 └──── difference ───┘
                           │
                           ↓
                    update student
```

### 为什么 DMD 和 Self-Forcing 很搭？

Self-Forcing 决定 sample 从哪里来：

$$
\hat x_{1:T}\sim p_\theta^{AR}.
$$

DMD 决定如何优化这些 on-policy samples：

$$
p_\theta(\hat x_{1:T})
\rightarrow
p_T(x_{1:T}).
$$

所以：

- **SF**：on-policy rollout mechanism；
- **DMD**：distribution-level training objective。

---

## 三条代表路线放在一起看

如果只选三个代表模型，可以比较：

$$
\boxed{
\text{Causal-rCM}
\quad vs\quad
\text{MAGI-1}
\quad vs\quad
\text{InfinityStar}
}
$$

分别对应：

- strong bidirectional teacher -> distilled causal AR；
- native continuous causal AR diffusion；
- native discrete spacetime next-scale AR。

| 维度 | Causal-rCM | MAGI-1 | InfinityStar |
| --- | --- | --- | --- |
| AR 从哪里来 | post-training / distillation | native causal pretraining | native AR pretraining |
| representation | continuous VAE latent | continuous VAE latent | discrete BSQ tokens |
| temporal AR unit | frame / chunk | chunk | clip |
| unit 内部过程 | few-step diffusion / flow | diffusion / flow | next-scale residual prediction |
| diffusion NFE | yes，目标压到 1–few step | yes，base model 可 many-step | no diffusion NFE |
| spatial coarse-to-fine AR | no | no | yes |
| teacher | causal teacher + DMD teacher/scorer | pretraining 可独立完成 | generator 训练不依赖 diffusion teacher |
| exposure-bias handling | SF-DMD | native causal training / noisy history structure | Bitwise Self-Correction |
| tokenizer bottleneck | continuous VAE reconstruction | continuous VAE reconstruction | discrete quantization + tokenizer reconstruction |
| streaming | frame/chunk-wise | chunk-wise | clip-wise |
| 核心优势 | 复用强 diffusion foundation model | causality 从 pretraining 建立 | discrete AR、next-scale efficiency、统一 image/video |

---

## 一个统一的理解方式

### Distilled AR 在解决什么？

已有一个很强的 bi-directional diffusion teacher，希望快速得到：

$$
\text{causal}
+
\text{streaming}
+
\text{few-step}.
$$

主要困难依次包括：

1. architecture / information-boundary mismatch；
2. many-step -> few-step initialization；
3. exposure bias；
4. DMD scorer 是否使用 future information；
5. long-horizon error accumulation。

### Native continuous AR 在解决什么？

从 pretraining 阶段就使用 causal temporal structure：

$$
p(x_t\mid x_{<t}).
$$

重点变成：

- 如何 scale causal diffusion pretraining；
- 如何让 noisy context 稳定；
- 如何提高 chunk parallelism；
- 如何继续把 NFE 降下来。

### Native discrete next-scale AR 在解决什么？

希望直接使用 GPT-like discrete prediction：

$$
p(r_{c,k}\mid\text{past}).
$$

重点变成：

- visual tokenizer reconstruction；
- coarse-to-fine factorization；
- bitwise vocabulary scaling；
- temporal clip consistency；
- cross-scale / cross-clip attention cost；
- AR error correction。

---

## 推荐阅读顺序

如果目标是理解「AR video 为什么会从 diffusion 走到现在」，建议按下面顺序：

1. **VAR**：先理解 next-scale prediction；
2. **Infinity**：理解 bitwise tokenizer / classifier 和 Bitwise Self-Correction；
3. **InfinityStar**：理解 spacetime pyramid；
4. **Diffusion Forcing**：理解 causal diffusion 的二维 time/noise view；
5. **MAGI-1**：看 native causal diffusion 如何 scale 到大型视频模型；
6. **CausVid**：看 bidirectional diffusion 如何改成 streaming AR；
7. **Self-Forcing**：理解 exposure bias；
8. **Causal Forcing / Causal Forcing++**：理解 causal teacher 和 few-step initialization；
9. **Causal-rCM**：理解 TF-CM + SF-DMD 的组合；
10. **CMD**：理解 teacher scoring 的 information-set matching。

---

## References

- VAR: *Visual Autoregressive Modeling: Scalable Image Generation via Next-Scale Prediction*, arXiv:2404.02905
- Diffusion Forcing: *Diffusion Forcing: Next-token Prediction Meets Full-Sequence Diffusion*, arXiv:2407.01392
- Infinity: *Infinity: Scaling Bitwise AutoRegressive Modeling for High-Resolution Image Synthesis*, arXiv:2412.04431
- CausVid: *From Slow Bidirectional to Fast Autoregressive Video Diffusion Models*, arXiv:2412.07772
- MAGI-1: *MAGI-1: Autoregressive Video Generation at Scale*, arXiv:2505.13211
- Self-Forcing: *Self Forcing: Bridging the Train-Test Gap in Autoregressive Video Diffusion*, arXiv:2506.08009
- InfinityStar: *InfinityStar: Unified Spacetime AutoRegressive Modeling for Visual Generation*, arXiv:2511.04675
- Causal Forcing: *Causal Forcing: Autoregressive Diffusion Distillation Done Right for High-Quality Real-Time Interactive Video Generation*, arXiv:2602.02214
- Causal Forcing++: *Causal Forcing++: Scalable Few-Step Autoregressive Diffusion Distillation for Real-Time Interactive Video Generation*, arXiv:2605.15141
- Causal-rCM: *Causal-rCM: A Unified Teacher-Forcing and Self-Forcing Open Recipe for Autoregressive Diffusion Distillation in Streaming Video Generation and Interactive World Models*, arXiv:2606.25473
- CMD: *Context-Matched Distillation: Teacher Causality for Autoregressive Video Distillation*, arXiv:2608.13391
- DMD: *One-step Diffusion with Distribution Matching Distillation*, CVPR 2024 / arXiv:2311.18828
