# 附录：示例 → 主题速查

> [← 回到目录](README.md)

按目录列出示例，方便按主题跳转回对应章节。

## 应用与 ECS
- **应用骨架**：`examples/app/{empty,empty_defaults,plugin,plugin_group,custom_loop,return_after_run,headless,without_winit}.rs` → [01](01-app-skeleton.md)
- **ECS 总览**：`examples/ecs/{ecs_guide,startup_system,system_param,system_piping,generic_system,one_shot_systems,callbacks,custom_schedule,custom_executor,run_conditions,change_detection,delayed_commands,error_handling,fallible_params}.rs` → [02](02-ecs-core.md)
- **查询**：`examples/ecs/{parallel_query,contiguous_query,iter_combinations,custom_query_param,immutable_components}.rs` → [02](02-ecs-core.md)
- **消息/观察者**：`examples/ecs/{message,observers,observer_propagation,removal_detection,component_hooks}.rs` → [02](02-ecs-core.md)、[14](14-per-entity-state-and-derived-stats.md)
- **状态**：`examples/state/{states,sub_states,computed_states,custom_transitions}.rs`、`examples/ecs/state_scoped.rs` → [02](02-ecs-core.md)、[14](14-per-entity-state-and-derived-stats.md)
- **反射**：`examples/reflection/{reflection,reflection_types,dynamic_types,generic_reflection,serialization,type_data}.rs` → [02](02-ecs-core.md)
- **场景**：`examples/scene/{bsn,world_serialization}.rs` → [03](03-scene-and-serialization.md)
- **时间**：`examples/time/{time,timers,virtual_time}.rs`、`examples/ecs/fixed_timestep.rs` → [01](01-app-skeleton.md)

## 数学与渲染
- **数学/Gizmos**：`examples/math/*`、`examples/gizmos/*` → [04](04-transform-math-gizmos.md)
- **变换**：`examples/transforms/*` → [04](04-transform-math-gizmos.md)
- **2D**：`examples/2d/*` → [05](05-2d-rendering.md)
- **3D**：`examples/3d/*`、`examples/gltf/*` → [06](06-3d-rendering.md)
- **动画**：`examples/animation/*` → [06](06-3d-rendering.md)
- **拾取**：`examples/picking/*`、`examples/3d/mesh_ray_cast.rs` → [06](06-3d-rendering.md)
- **着色器**：`examples/shader/*`、`examples/shader_advanced/*` → [10](10-shaders.md)

## UI 与交互
- **UI**：`examples/ui/*`（含 `ui/widgets`、`ui/text`、`ui/layout`、`ui/navigation`、`ui/scroll_and_overflow`、`ui/images`、`ui/styling`）→ [07](07-ui-and-text.md)
- **输入/窗口**：`examples/input/*`、`examples/window/*` → [08](08-input-and-window.md)

## 资源与工具
- **资产/异步/音频**：`examples/asset/*`、`examples/async_tasks/*`、`examples/audio/*` → [09](09-assets-async-audio.md)
- **工具链**：`examples/diagnostics/*`、`examples/dev_tools/*`、`examples/inspector/*`、`examples/remote/*`、`examples/tools/*` → [11](11-tooling-and-debugging.md)
- **移动/no_std**：`examples/mobile/*`、`examples/no_std/*` → [11](11-tooling-and-debugging.md)
- **压测/测试台/展示**：`examples/stress_tests/*`、`examples/testbed/*`、`examples/showcase/*` → [12](12-performance-stress-tests.md)
- **用法/辅助**：`examples/usage/*`、`examples/helpers/*` → [14](14-per-entity-state-and-derived-stats.md)（数据建模）、[03](03-scene-and-serialization.md)（BSN）
