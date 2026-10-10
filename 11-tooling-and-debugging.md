# 11. 工具链与调试（诊断 / 检查器 / 远程 / 工具 / 移动 / no_std）

> [← 回到目录](README.md)

## 11.1 诊断（Diagnostics）

### 【方案】
```rust
// examples/diagnostics/log_diagnostics.rs
App::new().add_plugins((
    DefaultPlugins,                        // 必须在前（依赖 Time）
    LogDiagnosticsPlugin::default(),
    FrameTimeDiagnosticsPlugin::default(), // FPS/FRAME_TIME/FRAME_COUNT
    EntityCountDiagnosticsPlugin::default(),
    SystemInformationDiagnosticsPlugin,    // 进程/系统 CPU & 内存
    bevy::render::diagnostic::RenderDiagnosticsPlugin,
))
// 自定义诊断（custom_diagnostic.rs）
.register_diagnostic(Diagnostic::new(PATH).with_suffix(" iterations"))
fn my_system(mut diagnostics: Diagnostics) { diagnostics.add_measurement(&PATH, || 10.0); }
```

### 【原理】
`FrameTimeDiagnosticsPlugin` 在 `Update` 用 `Time<Real>::delta_secs_f64()` 算 `fps=1/Δ`。每个 `Diagnostic` 维护历史、`sum`、`ema`（指数移动平均，`ema_smoothing_factor` 默认 `2/(N+1)`）。`SystemInformationDiagnosticsPlugin` 因采集耗时，在后台任务执行。`Diagnostics` 是延迟写入的 SystemParam。

### 【最佳实践】
诊断插件加在 `DefaultPlugins` 之后。想画自己的 HUD 就别加 `LogDiagnosticsPlugin`，直接读 `DiagnosticsStore`。诊断必须先注册再测量。

---

## 11.2 开发工具（Dev Tools）

### 【方案】
FPS 悬浮层：`FpsOverlayPlugin { config: FpsOverlayConfig { text_config, text_color, refresh_interval, frame_time_graph_config, .. } }`（`fps_overlay.rs`）。无限网格：`InfiniteGridPlugin` + `commands.spawn(InfiniteGrid)`（自定义 WGSL 屏幕空间绘制，编辑器地面参考）。调度数据导出：`SerializeSchedulesPlugin` → RON（`schedule_data.rs`）。`easy_camera`（`FreeCamera`）/`easy_screenshot`。

### 【最佳实践】
仅开发/CI 启用这些。`refresh_interval < 50ms` 会告警（可能掉性能）。

---

## 11.3 检查器（Inspector）

### 【问题】
类似引擎编辑器的面板，浏览实体树、查看并按字段编辑任意（已反射）组件/资源。

### 【方案】
```rust
// examples/inspector/local_inspector.rs
App::new()
    .add_plugins((DefaultPlugins, FeathersPlugins, InspectorPlugin))
    .insert_resource(UiTheme(create_dark_theme()))
    .add_systems(Startup, (demo_scene.spawn(), inspector_ui.spawn()));
// 远程：.insert_resource(InspectorSource::Remote(remote_source()))
```

### 【原理】
`InspectorPlugin` 初始化 `InspectorSource`/`InspectorSelection` 等资源；实体树按 `ChildOf/Children` 构建；编辑经事件 `FieldEdit → apply_field_edit → write_field_edit`，从 `AppTypeRegistry` 取 `ReflectComponent` 写字段。`Local` source 可写，`Remote` source 只能观察。

### 【最佳实践】
需要 `bevy_inspector`（+`debug` 取名）feature。要检查/编辑的组件必须 `Reflect`，资源需 `#[reflect(Resource)]`。不可变组件无法编辑。

---

## 11.4 远程（BRP：Bevy Remote Protocol）

### 【问题】
不改目标进程代码，从外部（终端/测试框架/检查器）查询和修改正在运行的 App。

### 【方案】
```rust
// examples/remote/server.rs（被检查方）
App::new().add_plugins((DefaultPlugins, RemotePlugin::default(), RemoteHttpPlugin::default()))
// 可被远程读写的类型：Reflect + Serialize + Deserialize + #[reflect(Component/Resource)]
```
内建方法：`world.query`/`get_components`/`spawn_entity`/`insert_components`/`mutate_components`/`despawn_entity`；`world.get_resources`/`insert_resources`/`mutate_resources`；`world.trigger_event`/`write_message`；`+watch`（SSE 流）；`world.inspect*`/`registry.schema`；`app.info`/`diagnostics.*`。

### 【原理】
`RemoteHttpPlugin` 用 hyper+smol 起 HTTP/1 服务，客户端 POST JSON 到根 URL。`+watch` 用 SSE 推送。可用 `write_message` 注入输入事件远程驱动交互（`integration_test.rs` 就是这么点按钮、截图、断言的）。

### 【最佳实践】
类型必须 `Reflect + Serialize + Deserialize`。窗口必须可见否则截图全黑。`RemotePlugin`+`RemoteHttpPlugin` 只加在开发/测试构建，**不要暴露到发布版（有安全风险）**。

---

## 11.5 工具（Tools）

- glTF 场景查看器（`tools/scene_viewer/`）：`argh` 解析参数 + `SceneViewerPlugin` 加载后按 AABB 估算包围盒设相机，插入 `DepthPrepass`/`OcclusionCulling`/`MotionBlur` 等。
- 手柄可视化（`tools/gamepad_viewer.rs`）：Sprite 画按钮/摇杆/触发器，调试手柄映射。

---

## 11.6 移动（Mobile）/ no_std

- 移动（`examples/mobile/`）：`#[bevy_main]` 入口宏；`WinitSettings::mobile()` 降耗电；触摸用 `TouchInput`/`RotationGesture`；生命周期 `AppLifecycle`（`Suspended→pause`）；对脆弱平台用 `#[cfg(target_os="android")]` 关闭会崩的特性（点光阴影/MSAA）。
- no_std（`examples/no_std/`）：`#![no_std]` + `extern crate alloc`；用 `bevy::platform::{collections, sync, time}` 而非直接 std；features 暴露 `std`/`libm`/`critical-section`；lints 强制 `std_instead_of_core`。

---

## 相关章节
- 反射是检查器/远程的基础：[ECS 核心 · Reflection](02-ecs-core.md#26-reflection反射)
- 性能测量：[性能与压测](12-performance-stress-tests.md)
