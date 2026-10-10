# 12. 性能与压测

> [← 回到目录](README.md)

`examples/stress_tests/`（须用 `--profile stress-test` 运行）针对不同子系统的性能：

| 示例 | 压测目标 |
|---|---|
| `bevymark.rs` / `bevymark_3d.rs` | 海量精灵基准，考察批处理/渲染吞吐 |
| `many_sprites.rs` | 大量 Sprite；`--colored` 破坏批处理以对比 |
| `many_cubes.rs` | 逐实体 draw 开销；材质/实例化参数 |
| `many_foxes.rs` | 大量蒙皮 glTF 动画 |
| `many_lights.rs` / `many_cameras_lights.rs` | 多光源/多相机（WebGL2 受 uniform 限制，最多 256 灯） |
| `many_materials.rs` | 大量材质特化开销；`--unique-materials` |
| `many_gizmos.rs` / `many_glyphs.rs` / `many_text*.rs` | gizmo / 文本渲染 |
| `many_buttons.rs` | UI 布局+文本+交互+渲染综合 |
| `many_gradients.rs` | UI 渐变（验证 CPU 预转换到目标色彩空间） |
| `many_morph_targets.rs` | 形变目标动画 |
| `many_components.rs` | 超大 ECS 世界 |
| `transform_hierarchy.rs` | 层级变换传播 |

## 【原理】
这些示例量化 Bevy 各子系统的批处理、实例化、剔除、管线特化、变更检测开销。

## 【最佳实践】
必须用 release/stress-test profile 运行，否则结果无意义。用它对照大场景示例的 `--no-*` 开关定位瓶颈。

## 相关章节
- 大场景的 `--no-*` 开关：[3D 渲染 · 大场景](06-3d-rendering.md)
- 跨模块经验总表：[最佳实践与陷阱速查](13-best-practices-cheatsheet.md)
