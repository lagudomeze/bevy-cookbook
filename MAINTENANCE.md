# 维护与更新流程

本手册是「派生文档」：内容来自 Bevy 本体的 `examples/` 与 `crates/`，但本身是独立仓库。
Bevy 演进很快（API 频繁改名），因此**每次更新都应记录来源 commit**。

## 版本锚点

当前生成自：`bevy@0e93c7ab7`（2026-10-10）。
每次重扫后，更新 [README.md](README.md) 顶部那行 commit 与日期。

## 更新步骤

1. **更新本地 Bevy 检出**（不是本仓库）：
   ```bash
   cd /path/to/bevy
   git fetch origin && git pull --rebase origin main
   git rev-parse --short HEAD          # 记下新 commit
   ```
2. **重扫示例**：对照下面「章节 → 源码」表，逐章比对 `examples/`、`crates/` 是否变化。
   重点关注：新增/删除的示例、改名的 API、迁移的目录。
3. **更新文档**：改对应章节（保持「问题 → 方案 → 原理 → 最佳实践」四段式），更新交叉链接。
4. **更新版本锚点**：把 README 顶部的 commit/日期改成新的。
5. **提交并推送**（本仓库）：
   ```bash
   cd cookbook
   git add -A && git commit -m "rescan: bevy@<new-sha>"
   git push
   ```

## 章节 → 源码映射（用于定向重扫）

| 章节 | 对应 `examples/` | 对应 `crates/` |
|---|---|---|
| 01 应用骨架 | `examples/app/`、`examples/ecs/`（调度/时间）| `bevy_app` |
| 02 ECS 核心 | `examples/ecs/`、`examples/state/`、`examples/reflection/` | `bevy_ecs`、`bevy_state`、`bevy_reflect` |
| 03 场景与序列化 | `examples/scene/` | `bevy_scene`、`bevy_world_serialization` |
| 04 变换/数学/Gizmos | `examples/transforms/`、`examples/math/`、`examples/gizmos/` | `bevy_transform`、`bevy_math`、`bevy_curve`、`bevy_gizmos` |
| 05 2D 渲染 | `examples/2d/`、`examples/camera/` | `bevy_sprite`、`bevy_camera` |
| 06 3D 渲染 | `examples/3d/`、`examples/gltf/`、`examples/animation/`、`examples/picking/`、`examples/large_scenes/` | `bevy_pbr`、`bevy_gltf`、`bevy_animation`、`bevy_picking`、`bevy_render` |
| 07 UI 与文本 | `examples/ui/` | `bevy_ui`、`bevy_text`、`bevy_ui_widgets`、`bevy_feathers` |
| 08 输入与窗口 | `examples/input/`、`examples/window/` | `bevy_input`、`bevy_window`、`bevy_winit` |
| 09 资产/异步/音频 | `examples/asset/`、`examples/async_tasks/`、`examples/audio/` | `bevy_asset`、`bevy_tasks`、`bevy_audio` |
| 10 着色器 | `examples/shader/`、`examples/shader_advanced/`、`assets/shaders/` | `bevy_shader`、`bevy_render` |
| 11 工具链与调试 | `examples/diagnostics/`、`examples/dev_tools/`、`examples/inspector/`、`examples/remote/`、`examples/tools/`、`examples/mobile/`、`examples/no_std/` | `bevy_diagnostic`、`bevy_dev_tools`、`bevy_inspector`、`bevy_remote` |
| 12 性能与压测 | `examples/stress_tests/`、`examples/testbed/`、`examples/showcase/` | — |
| 13 速查表 | （综合）| — |
| 14 每实体状态与派生数值 | `examples/ecs/observers.rs`、`removal_detection.rs`、`change_detection.rs`、`component_hooks.rs` | `bevy_ecs` |
| 00 版本与 API 警示 | 全仓库 | 全仓库 |

## 已知易变点（重扫时优先核对）

- **API 改名**：缓冲事件 `Message` vs 观察者 `Event`；场景 `WorldAssetRoot`；`GlobalAmbientLight`；`shadow_maps_enabled`；着色器 `.wesl`；`DespawnOnExit` 等。见 [00-version-warning.md](00-version-warning.md)。
- **UI**：`bevy_ui_widgets` / `bevy_feathers` 仍在快速演进。
- **BSN**：`.bsn` 文件格式尚未发布；`bsn!` 宏语法可能变。
- **引用路径**：bevy 会移动示例目录，旧路径会 404 —— 重扫时用 `grep` 验证引用是否仍存在。
