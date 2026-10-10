# 13. 最佳实践与陷阱速查

> [← 回到目录](README.md)

## 跨模块通用
- 逻辑改 `Transform`，读世界用 `GlobalTransform`；读传播结果的系统排在 `TransformSystems::Propagate` 之后。
- 位置/旋转更新永远乘 `delta_secs()`；求夹角先 `clamp(-1,1)`。
- 会阻塞的 IO 交给 `IoTaskPool`；任务结果用 `check_ready` 轮询，别 `block_on`。
- 高频多消费者用 `Message`；立即定向响应用 `Observer`。
- 变更检测加 `Changed<T>`/`Added<T>`；避免无意义触发用 `set_if_neq`。

## UI 三大陷阱（`examples/ui/` 高频）
1. **尺寸读取**：声明值在 `Node`（`Val`），像素值只在 `ComputedNode`，且要乘 `inverse_scale_factor`。
2. **多相机/多窗口**：UI 不认 `RenderLayers`，必须 `UiTargetCamera`。
3. **控件无样式**：`bevy_ui_widgets` 只给行为，外观自备；要现成主题用 `bevy_feathers`。

## 渲染陷阱
- 要发光必须给颜色分量 > 1，且相机加 `Hdr`。
- 要自动实例化，所有实例共享同一 mesh+material handle。
- 不透明物别设 `AlphaMode::Blend`；大量树叶用 `Mask`。
- 线框/OIT 在移动端/WebGL 常不可用，做好降级。

## ECS 陷阱
- 同系统 `MessageReader` + `MessageWriter` 同类型冲突 → 用 `MessageMutator`。
- 命令延迟生效，同系统 `spawn` 后立刻 `query` 拿不到新实体。
- 独占系统会拖垮并行度，非必要不用。
- 依赖 `SubStates`/`ComputedStates` 的系统在状态不存在时用 `run_if(state_exists::<S>)`。
- `States` 是全局的，别拿它表达「单个实体的状态」——那是普通组件。

## 数据建模陷阱
- 「派生数值」别每帧重算，用脏标记 + `run_if`；修正项**先加算后乘算**。
- 详见 [每实体状态与派生数值](14-per-entity-state-and-derived-stats.md)。

## 相关章节
- 全部章节见 [目录](README.md)
