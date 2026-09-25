# 数据说明

本页记录候选训练与评测资源；尚未下载机器人数据、模型权重或运行实验。

| 候选资源 | 主要用途 | 本地状态 | 官方入口 |
|---|---|---|---|
| LIBERO | 初步策略复现、任务闭环 | 未下载 | [官方仓库](https://github.com/Lifelong-Robot-Learning/LIBERO) |
| LIBERO-Plus | 相机等扰动评测 | 未下载 | [官方仓库](https://github.com/sylvestf/LIBERO-plus) |
| RoboTwin 2.0 | 双臂合成数据、域随机化、多相机 | 未下载 | [文档](https://robotwin-platform.github.io/doc/) |
| TriWorldBench | 三视角视频生成评测 | 未下载；验证/测试真值权限须分别确认 | [官方仓库](https://github.com/TriWorldBench/TriWorldBench) |
| FastWAM 官方预处理数据 | 对齐原论文数据管线 | 未下载；使用前固定版本与统计文件 | [官方仓库](https://github.com/yuantianyuan01/FastWAM) |

正式下载时追加：许可、来源URL/版本、文件大小/hash、原始目录结构、任务/轨迹/视角数量、过滤规则、时间同步、图像裁剪、动作类型/坐标系/单位/归一化、划分清单。禁止用测试集计算归一化统计。

划分原则见 [evaluation.md](evaluation.md)：按目标泛化维度设计，相同episode/state的其他视角不能无意进入另一split；相机内插/外推与新场景泛化要分别声明。
