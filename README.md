# Lens-Free Camera

无镜头散射成像的数值仿真与三维重建演示。公开仓库仅保存说明、演示图、动画和结果数据，不包含核心实现代码。

**语言 / Language:** [简体中文](README.zh-CN.md) · [English](README.en.md)

## 快速查看 / At a glance

| 演示 / Demo | 文件 / File |
| --- | --- |
| 光学与反演流程 / Optical and inverse pipeline | [中文图 / ZH](assets/optical-path.zh-CN.svg) · [EN](assets/optical-path.en.svg) |
| 深度层重建 / Depth-layer reconstruction | [中文图 / ZH](assets/reconstruction-overview.zh-CN.png) · [EN](assets/reconstruction-overview.en.png) |
| 多 seed 稳定性 / Multi-seed stability | [中文图 / ZH](assets/depth-error-reduction.zh-CN.png) · [EN](assets/depth-error-reduction.en.png) |
| 动画演示 / Animated demo | [中文 GIF / ZH](assets/demo.zh-CN.gif) · [EN GIF](assets/demo.en.gif) |
| 可复用结果 / Reusable results | [NPZ 数据 / Dataset](data/meeting_scene_results.npz) |

## 结果摘要 / Result summary

默认 seed 7 的合成会议室场景包含 1,920 个候选体素，分布在 6 个深度层。96×96 传感器图像有 8,136 个有效采样点。启用残差门控的 6 轮深度精修后，相对体积误差为 `0.249`，深度 argmax 准确率为 `0.944`，有效传感器像素上的相对拟合残差为 `0.0163`。

These are synthetic-scene metrics, not measurements from a physical camera. The complete setup and limitations are documented in both language guides.

## 公开资料 / Public materials

- [中文技术说明](docs/technical-notes.zh-CN.md) · [English technical notes](docs/technical-notes.en.md)
- [中文实验与指标](docs/validation.zh-CN.md) · [English evaluation notes](docs/validation.en.md)
- [中文逐 seed CSV](data/depth_stability_sweep.zh-CN.csv) · [English per-seed CSV](data/depth_stability_sweep.en.csv)
- [完整会议室重建图](assets/meeting_scene_results.png)

深度精修仅适用于不透明单表面先验，是局部坐标更新，不保证全局最优。高残差门控在当前合成压力测试中会跳过不可靠的深度交换；阈值需要针对真实传感器重新校准。
