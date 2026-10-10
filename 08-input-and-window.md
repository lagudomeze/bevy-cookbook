# 08. 输入与窗口

> [← 回到目录](README.md)

## 8.1 键盘

### 【方案】
`Res<ButtonInput<KeyCode>>` 提供 `pressed/just_pressed/just_released`；`Res<ButtonInput<Key>>` 按字符。**`KeyCode`** 是物理键位（跨布局稳定，适合 WASD）；**`Key`** 是具体字符（适合 `?`/`+/-`）。原始事件 `MessageReader<KeyboardInput>`（含 `key_code`/`logical_key`/`state`）。修饰键 `input.any_pressed([ShiftLeft, ShiftRight])`。示例：`examples/input/keyboard_input*.rs`。

### 【原理】
`ButtonInput<T>` 是每帧聚合资源；原始 `KeyboardInput` 消息保留更多信息（`repeat`/`window`）。

### 【最佳实践】
游戏操作优先 `KeyCode`，文字输入优先 `Key`/字符事件。修饰键要同时考虑左右两键。

---

## 8.2 鼠标 / 触摸 / 手柄

- 鼠标：`Res<ButtonInput<MouseButton>>`；`Res<AccumulatedMouseMotion>`（本帧累计 delta，适合 FPS 视角）；`Res<AccumulatedMouseScroll>`；原始事件 `MouseButtonInput`/`MouseMotion`/`CursorMoved`/`MouseWheel`（`examples/input/mouse_input*.rs`）。
- 触摸：`Res<Touches>` 按 finger `id` 跟踪多指，`iter_just_pressed/released/canceled`；原始 `MessageReader<TouchInput>`（`TouchPhase::{Started,Moved,Ended,Canceled}`）。
- 手柄（bevy_gilrs）：每个手柄一个带 `Gamepad` 组件的实体，`gamepad.just_pressed(GamepadButton::South)`，模拟量 `gamepad.get(GamepadAxis::LeftStickX)`；震动 `GamepadRumbleRequest::Add { intensity: strongly/weak, duration }`（`gamepad_rumble.rs`）。

### 【原理】
`AccumulatedMouseMotion/Scroll` 把一帧多个事件合并成单个 `delta`，避免丢帧；`MouseMotion` 是设备原始位移（不受窗口边界影响），`CursorMoved` 是窗口坐标。手柄由 gilrs 后端接入，布尔「按下」由 `GamepadSettings` 阈值判定。

### 【最佳实践】
视角控制用 `AccumulatedMouseMotion`（不乘 `delta_secs`）。滚轮区分 `Line`（乘行高）/`Pixel`。无手柄时 `get` 返回 `None`，要兜底。

---

## 8.3 窗口：设置、多窗口、光标、截图、省电

### 【方案】
```rust
// examples/window/window_settings.rs
DefaultPlugins.set(WindowPlugin { primary_window: Some(Window {
    title: "I am a window!".into(), resolution: (500, 300).into(),
    present_mode: PresentMode::AutoVsync, window_theme: Some(WindowTheme::Dark),
    visible: false,  // 先隐藏，GPU 就绪后再显示，避免白屏
    ..default() }), ..default() })
```
- 多窗口：`commands.spawn(Window { .. })` + 相机 `RenderTarget::Window(WindowRef::Entity(id))` + `UiTargetCamera`。
- 光标：`CursorOptions { visible, grab_mode, hit_test }`、`CursorIcon::from(SystemCursorIcon::..)`、自定义图片光标、`window.start_drag_move()/start_drag_resize(CompassOctant::NorthWest)`。
- 截图：`commands.spawn(Screenshot::primary_window()).observe(save_to_disk(path))`。
- 省电：`WinitSettings::desktop_app()`（响应式）；游戏用 `WinitSettings::game()`。
- 设置持久化：`SettingsGroup` + `SettingsPlugin::new("org...")` + `exit_condition: ExitCondition::DontExit`（`persisting_window_settings.rs`）。

### 【原理】
每个 `Window` 是实体；相机经 `RenderTarget` 绑定窗口。`WinitSettings` 控制 winit 事件循环更新策略（游戏=连续，应用=有事件才更新）。UI 只随 `UiTargetCamera` 走，不认 `RenderLayers`。

### 【最佳实践】
`visible: false` + 第 3 帧再显示消除启动白屏。多窗口一定显式 `UiTargetCamera`。桌面工具用 `desktop_app()` 显著降 CPU，但 UI 动画需手动 `RequestRedraw`。

---

## 相关章节
- 输入驱动 UI 控件：[UI 与文本](07-ui-and-text.md)
- 窗口 resize 与分屏：[3D 渲染 · 分屏](06-3d-rendering.md)
