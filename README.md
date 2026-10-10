# Bevy 示例实战 Cookbook

> **基于 `bevy@0e93c7ab7` 生成（2026-10-10）** —— bevy 是快速演进的「移动靶」，本手册的结论绑定在该提交上；若你的 bevy 版本相距较远，请以你本地的 `examples/` 为准。更新流程见 [MAINTENANCE.md](MAINTENANCE.md)。

> 一份基于本仓库（`main` 分支）`examples/` 下 **444 个示例** 与相关 `crates/` 源码整理的实战手册。
> 官方文档按类型/函数罗列，不讲「我要实现 X 该用哪个模式」；本手册用 **问题 → 方案 → 原理 → 最佳实践** 四段式补齐这一层。

## 怎么用这份 cookbook

- **按需查阅**：不知道某个能力怎么做 → 到下面目录找对应章节。
- **四段式**：每节都按【问题】【方案】【原理】【最佳实践】组织，方案里给出可直接跳转的源码路径。
- **先读版本警示**：本仓库是「移动靶」，与网上 0.14/0.15 教程差异极大，务必先看 [00 版本与 API 警示](00-version-warning.md)。
- **只读源码**：本仓库示例默认需要窗口（GUI），很多在无头环境跑不起来；本手册全部基于读源码整理。

---

## 目录

### 入门与核心

| 章节 | 内容 |
|---|---|
| [00. 版本与 API 警示](00-version-warning.md) | 本仓库相对旧教程的改名对照（`Message`/`WorldAssetRoot`/`bsn!`/`.wesl`…），**先读这个** |
| [01. 应用骨架](01-app-skeleton.md) | App / Plugin / Schedule / System / SystemSet / Time / FixedUpdate / Timer |
| [02. ECS 核心](02-ecs-core.md) | 查询 / 命令 / 资源 / 消息 / 观察者 / 状态 / 反射 |
| [03. 场景与序列化](03-scene-and-serialization.md) | BSN 声明式场景 / 世界序列化 / 存档 |
| [14. 每实体状态与派生数值](14-per-entity-state-and-derived-stats.md) | **per-entity 状态机**、**数值修正/派生属性缓存**（buff/装备/基础值） |

### 数学与渲染

| 章节 | 内容 |
|---|---|
| [04. 变换、数学与 Gizmos](04-transform-math-gizmos.md) | Transform/GlobalTransform / 移动旋转 / 曲线 / 包围体 / 调试绘制 |
| [05. 2D 渲染](05-2d-rendering.md) | Sprite / 图集 / Mesh2d / Bloom / Text2d / 2D 相机与坐标转换 |
| [06. 3D 渲染](06-3d-rendering.md) | 相机 / PBR / 光照 / 几何 / 雾大气 / 后处理 / 透明 / 实例化 / glTF / 动画 / 拾取 / 大场景 |
| [10. 着色器](10-shaders.md) | WESL / 自定义材质 / 存储缓冲 / GPU 读回 / 计算着色器 / 渲染图 |

### UI、交互与资源

| 章节 | 内容 |
|---|---|
| [07. UI 与文本](07-ui-and-text.md) | 布局 / 滚动 / 控件 / 文本输入 / 焦点导航 / UI 拾取 / 缩放 |
| [08. 输入与窗口](08-input-and-window.md) | 键盘 / 鼠标 / 触摸 / 手柄 / 多窗口 / 光标 / 截图 / 省电 |
| [09. 资产、异步与音频](09-assets-async-audio.md) | AssetServer/Handle / 热重载 / 加载器 / 任务池 / 音频 |

### 工具、性能与速查

| 章节 | 内容 |
|---|---|
| [11. 工具链与调试](11-tooling-and-debugging.md) | 诊断 / dev tools / 检查器 / BRP 远程 / 工具 / 移动 / no_std |
| [12. 性能与压测](12-performance-stress-tests.md) | `stress_tests/` 各示例压测什么、怎么定位瓶颈 |
| [13. 最佳实践与陷阱速查](13-best-practices-cheatsheet.md) | 跨模块的坑与经验总表 |
| [附录：示例 → 主题速查](appendix-example-index.md) | 按目录列出的示例索引 |

---

## 约定

- 路径都相对仓库根，例如 `examples/ecs/ecs_guide.rs`。
- 代码片段是从示例里摘录/压缩的，可能省略了 import 与上下文。
- 带「⚠️ 版本警示」的章节说明该模块与旧版教程差异较大。
