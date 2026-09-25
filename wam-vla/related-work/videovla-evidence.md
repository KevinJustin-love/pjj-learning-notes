# VideoVLA v1 核验边界

- 核查版本：2512.06963v1，PDF 18 页；本笔记数字来自本地 PDF/TeX，不是把网页摘要扩写。
- §4 Implementation Details 写 DDIM 50 步；附录 D 写真机 10 步、单 H100 1.1 秒。保留两种配置，不自行断言每张真机表都采用同一个步数。
- 表 4 的 Place 含 Pick Up/Place 两阶段及其平均，所以 64.6 是论文的综合表平均，不能称全部试验最终成功率。
- 表 5 文本称 12 个新物体，但表列有 13 个配置列（含同瓶不同摆姿/不同瓶型）；因此笔记只引用表平均50.6，不将其包装成严格12类等权结果。
- 附录仿真新技能行统一写每技能20次，但表3存在6.3/56.3/58.3/93.8等与单一20次分母不相容的比例。可能涉及任务子条件聚合；论文未给足分母映射，不能用该表反推成功计数或置信区间。
- 不能根据名字接近建立引用关系：Fast-WAM `liang2025videogenerators`、DreamZero `liang2025video` 均指2508.00795 *Video Generators are Robot Policies*；确切的 VideoVLA 后续引用来自 LingBot-VA `shen2025videovla`。
