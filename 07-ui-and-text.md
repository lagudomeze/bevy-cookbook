# 07. UI 与文本

> [← 回到目录](README.md)
>
> ⚠️ **版本警示**：本仓库 UI 大改。`Text` 是普通组件（`Text::new`），多段文本用子实体 `TextSpan`；字体 API 改用 `TextFont { font: FontSource, font_size: FontSize, weight, .. }`，尺寸支持 `px/percent/vw/vh/Rem`；文本后端从 cosmic-text 换成 **parley + swash**；`Button`/`Slider`/`Checkbox` 等**不再带样式**，迁到 headless 的 `bevy_ui_widgets`；`bevy_feathers` 提供主题化外观；UI 渲染在独立 crate `bevy_ui_render`；大量 UI 用 BSN 场景宏搭建。

## 7.1 Node 基础与 Flexbox 布局

### 【问题】
如何放矩形/上色/加边框/贴图？横竖排、对齐、留白、撑满？`AlignItems` 和 `JustifyContent` 区别？

### 【方案】
```rust
// examples/ui/layout/flex_layout.rs
Node {
    width: percent(100), height: percent(100),
    flex_direction: FlexDirection::Column,  // 主轴方向
    align_items: AlignItems::Center,         // 交叉轴
    justify_content: JustifyContent::Center, // 主轴
    row_gap: MARGIN, column_gap: MARGIN,
    ...default()
}
```
尺寸/约束：`width/min_width/max_width/flex_basis`（`Val::Auto/Percent/Px`）。单元素：`align_self`/`justify_self`。`FixedNode` 让节点脱离父布局（类似 `position: fixed`）。

### 【原理】
布局由 **taffy**（Rust Flexbox/Grid）计算：`crates/bevy_ui/src/layout/` 维护 `entity_to_taffy`，算完写回 **`ComputedNode`**（物理像素 `size`、`content_size`、`scroll_position`、`inverse_scale_factor`）。**只有布局后才知道真实尺寸**，运行时读尺寸必须查 `ComputedNode`。

### 【最佳实践】
忘设 `flex_direction` 默认 `Row`，容易与预期相反。读像素尺寸用 `ComputedNode`（`Node` 里是声明值 `Val`）。`margin: auto()` 是居中技巧。

---

## 7.2 Grid 布局、定位与 z-index

### 【方案】
Grid：`display: Display::Grid` + `grid_template_columns/rows`（`GridTrack::min_content()/flex(1.0)/px(20)`）+ gap，跨列 `grid_column: GridPlacement::span(2)`（`grid.rs`）。定位：`PositionType::Absolute` + `left/right/top/bottom`，`margin: auto` 居中（`anchor_layout.rs`）。z 序：`ZIndex`（同级）/`GlobalZIndex`（全局）。

### 【原理】
同样走 taffy 的 grid 分支。层序遍历（`crates/bevy_ui/src/stack.rs`）综合 `GlobalZIndex`/`ZIndex`/树顺序生成绘制顺序。

### 【最佳实践】
浮层（tooltip/模态）用 `GlobalZIndex(i32::MAX)`。拖动时临时抬高，松手还原。

---

## 7.3 溢出、裁剪与滚动

### 【方案】
```rust
// examples/ui/scroll_and_overflow/overflow.rs
Overflow::visible() / clip_x() / clip_y() / clip() / scroll() / scroll_x() / scroll_y()
```
滚动条：headless `Scrollbar { orientation, target, min_thumb_length }` + `ScrollbarThumb`（`scrollbars.rs`）。滚动核心是改 `ScrollPosition`（逻辑坐标），注意乘 `ComputedNode.inverse_scale_factor` 换算。

### 【原理】
`ScrollPosition` 是独立组件，布局/渲染时合进 `ComputedNode.scroll_position`。滚动条是纯行为组件，绑定 `target` 把 thumb 位置与目标 `content_size/size` 关联。

### 【最佳实践】
`Overflow::scroll_*` 只是「允许滚动」，真正滚还要驱动 `ScrollPosition`。`MouseScrollUnit::Line` 需乘行高变像素。嵌套滚动靠「消费 delta + 停止传播」。

---

## 7.4 控件（Button / Checkbox / Slider / 下拉 / 列表…）

### 【问题】
怎么做按钮/滑块/复选框？headless 与主题化如何选？

### 【方案】
```rust
// examples/ui/widgets/button.rs
commands.spawn((Button, Hovered::default(), Node{..}, BackgroundColor(NORMAL_BUTTON),
    children![Text::new("Button")]))
    .observe(|_activate: On<Activate>| { info!("Button clicked!"); });
```
`bevy_ui_widgets` 提供纯行为组件：`Button`/`Checkbox`/`RadioButton`+`RadioGroup`/`Slider`+`SliderThumb`+`SliderValue`+`SliderRange`/`MenuButton`+`MenuItem`/`TabList`+`Tab`/`TreeView`/`SplitPane`/`Scrollbar`/`TextInput`/`VirtualKeyboard`；内置自更新系统 `checkbox_self_update`/`slider_self_update` 等；语义事件 `Activate`/`ValueChange<T>`。`bevy_feathers` 叠加上主题化外观（`FeathersPlugins`+`UiTheme`+`ThemeBackgroundColor(tokens::WINDOW_BG)`）。

### 【原理】
所有 headless 控件都是「指针事件 + 组件状态」的纯行为层：从 picking 的 `Hovered`/`Pointer*` 推导 `Pressed/Checked/Selected`，再发 `Activate/ValueChange`。feathers 在其上叠加视觉组件并通过 token 统一风格。

### 【最佳实践】
控件无样式是**有意设计**，必须自备 `Node/BackgroundColor`，否则「看不见」。用 `_self_update` 观察者最省心。`InteractionDisabled` 统一禁用。

---

## 7.5 文本与文本输入

### 【方案】
```rust
// examples/ui/text/text.rs
Text::new("hello\nbevy!"),
TextFont { font: asset_server.load("fonts/FiraSans-Bold.ttf").into(), font_size: FontSize::Vh(20.0), ..default() },
TextColor(GOLD.into()), TextLayout::justify(Justify::Center),
```
多段样式：父 `Text` + 子 `TextSpan`。文本输入（`text_input.rs`）：`TextInput` + `EditableText { visible_width, allow_newlines, .. }` + `TextCursorStyle` + `TabIndex`。字符过滤：`EditableTextFilter::new(|c| c.is_ascii_hexdigit())`。IME/CJK 靠系统字体（`FontSource::sans_serif()`）。

### 【原理】
`bevy_text` 流水线（`crates/bevy_text/src/pipeline.rs`）用 **parley** 对 `TextSpan` 做形状化得 `Layout`，用 **swash** 光栅化进 `FontAtlas`，产出 `TextLayoutInfo`。文本节点尺寸参与 UI 布局。

### 【最佳实践】
文本节点**不能直接加 margin/padding**，需包一层 `Node` 容器。缩放文字改 `font_size`。大量文本更新尽量只改内容（`**span = ...`）。

---

## 7.6 边框/阴影/渐变、焦点导航、UI 拾取

- 边框/阴影/渐变（`borders.rs`/`box_shadow.rs`/`gradients.rs`）：`BorderColor`+`Node.border`+`BorderRadius`、`Outline`、`BoxShadow(vec![ShadowStyle{..}])`+`BoxShadowSamples`、`BackgroundGradient::from(LinearGradient/RadialGradient/ConicGradient)`。
- 焦点导航：`TabNavigationPlugin` + `TabGroup`/`TabIndex`，`InputFocus` 存焦点实体；方向导航用 `DirectionalNavigationPlugin` + `AutoDirectionalNavigation`（`directional_navigation.rs`）。
- UI 拾取：`Hovered` 组件、`On<PointerOver>/<PointerDragStart>/<PointerDrag>/<PointerDragDrop>`；`Pickable::IGNORE` 让点击穿透；`RelativeCursorPosition` 提供归一化坐标。

### 【原理】
边框/阴影/渐变在 `bevy_ui_render` 的提取阶段分别抽取，在 shader 里用 SDF/半球采样实现圆角与模糊。方向导航用元素屏幕矩形按指南针八方向找最近邻（`crates/bevy_ui/src/auto_directional_navigation.rs`）。

---

## 7.7 UI 缩放、UiTransform、与 3D 混合

- UI 缩放：`UiScale` 资源全局缩放（`ui_scaling.rs`）；`WindowResolution::with_scale_factor_override`；读像素尺寸统一乘 `ComputedNode.inverse_scale_factor`。
- `UiTransform`：渲染期平移/旋转/缩放，**不参与布局**（`ui_transform.rs`）——适合抖动/脉冲/拖拽跟随等纯视觉动画。
- 3D 嵌 UI：`ViewportNode::new(camera_entity)`；UI 贴 3D：UI 相机渲染到纹理 + 材质用该纹理，点击靠射线→UV→虚拟指针事件（`render_ui_to_texture.rs`）；多相机 UI 用 `UiTargetCamera`；自定义 UI shader 用 `UiMaterialPlugin` + `MaterialNode`。

### 【原理】
UI 每个 camera 只渲染一棵 UI 树、只有一个 render target，**不支持 `RenderLayers`**。

### 【最佳实践】
多相机/多窗口场景必须显式 `UiTargetCamera`。像素位移/滚动/拖拽逻辑都要考虑 `inverse_scale_factor`。

---

## UI 三大陷阱
1. **尺寸读取**：声明值在 `Node`（`Val`），像素值只在 `ComputedNode`，且要乘 `inverse_scale_factor`。
2. **多相机/多窗口**：UI 不认 `RenderLayers`，必须 `UiTargetCamera`。
3. **控件无样式**：`bevy_ui_widgets` 只给行为，外观自备；要现成主题用 `bevy_feathers`。

## 相关章节
- 输入（键盘/手柄驱动 UI）：[输入与窗口](08-input-and-window.md)
- UI 控件与状态联动：[每实体状态与派生数值](14-per-entity-state-and-derived-stats.md)
- 用 BSN 搭 UI：[场景与序列化](03-scene-and-serialization.md)
