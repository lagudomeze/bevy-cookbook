# 00. 版本与 API 警示

> [← 回到目录](README.md)

## 【问题】

网上大量教程、博客基于 Bevy 0.14/0.15，而本仓库是 `main` 分支的开发版，**API 差异极大**，照搬旧教程会直接编译失败。开始写任何代码前，先建立「本仓库当前长这样」的认知。

## 【方案】关键改名对照表

| 概念 | 旧写法（0.14/0.15 常见） | 本仓库写法 |
|---|---|---|
| 缓冲事件 | `Event` / `EventReader` / `EventWriter` / `send` | **`Message` / `MessageReader` / `MessageWriter` / `write`**（读写同类型用 `MessageMutator`） |
| 观察者事件 | — | **`Event`（`On<T>`、`EntityEvent`）** / `commands.trigger` |
| 场景加载 | `SceneRoot` / `SceneSpawner` | **`WorldAssetRoot`** / `WorldInstanceReady` |
| 声明式场景 | — | **BSN：`bsn!` / `bsn_list!` / `spawn_scene` / `Children[...]` / `on(...)`** |
| 状态作用域 | `StateScoped` | **`DespawnOnExit` / `DespawnOnEnter` / `DespawnWhen`** |
| 环境光 | `AmbientLight` | **`GlobalAmbientLight`** |
| 阴影开关 | `shadows_enabled` | **`shadow_maps_enabled`** |
| 投影 | `Camera.projection` 直接字段 | **`Projection` 枚举（`::Perspective`/`::Orthographic`）** |
| 着色器 | `.wgsl` 内联 | **`.wesl`（WESL，带 `import` 模块系统）**；`assets/shaders/` 下无 `.wgsl` |
| 文本 | Bundle 化 `TextBundle` | **`Text` 是普通组件（`Text::new`）**，子实体 `TextSpan` |
| 字体 | `TextStyle` | **`TextFont { font: FontSource, font_size: FontSize, weight, .. }`**（parley+swash 后端） |
| 控件 | `Button` 带样式 | **`bevy_ui_widgets` headless 行为 + `bevy_feathers` 主题** |
| Handle 弱引用 | `WeakHandle` | **无 `Weak` 变体**；`Handle` 为 `Strong`/`Uuid` |
| 2D/3D 句柄 | `Handle<Mesh>` 直接作组件 | **`Mesh2d`/`Mesh3d`/`MeshMaterial2d`/`MeshMaterial3d` 新类型包装** |

## 【原理】为什么会有这么大差异

Bevy 把「缓冲事件」和「观察者事件」这两个语义完全不同的东西**分了家**：缓冲事件（多消费者、逐帧读、可延迟）改名为 `Message`；观察者事件（立即、定向、可冒泡）保留 `Event` 之名。场景/UI 则是一次较大的 API 重构（BSN 宏、headless 控件、新字体栈）。

## 【最佳实践】

- 遇到不确定的 API，**以 `examples/` 为准**，别信旧教程。
- `grep` 关键字：`grep -rn "关键字" examples/` 能找到权威用法。
- 拿不准某类型是否存在时，直接看 `crates/bevy_*/src/`。
- 本仓库示例默认需要 GUI 窗口，无头环境需自行判断能否运行。

## 相关文件

- 阅读指南与整体结构：[README](README.md)
- 消息 vs 观察者的用法：[ECS 核心 · 资源/消息/观察者](02-ecs-core.md#23-resourceslocalmessage缓冲事件observer观察者)
- WESL 着色器：[着色器](10-shaders.md)
