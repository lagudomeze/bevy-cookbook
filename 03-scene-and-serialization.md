# 03. 场景、BSN 与序列化

> [← 回到目录](README.md)

## 【问题】
如何把「可复用的实体组合」从代码/文件加载？世界序列化怎么存档？新的 BSN 宏长什么样？

## 【方案】
- 场景文件加载：`commands.spawn(DynamicWorldRoot(asset_server.load("...scn.ron")))`，加载后把文件内实体作为子级挂入（`examples/scene/world_serialization.rs`）。
- 保存：`DynamicWorld::from_world_with(&scene_world, &type_registry.read())` → `.serialize(&registry)` 得 RON，异步写文件。
- BSN（Bevy Scene Notation）声明式组合（`examples/scene/bsn.rs`）：
```rust
fn scene() -> impl SceneList { bsn_list! { Camera2d -- @ui() } }
fn ui() -> impl Scene {
    bsn! {
        Node { width: percent(100), .. }
        Children [
            @button("Ok") on(|_e: On<PointerPress>| println!("Ok!"))
            -- @button("Cancel") BackgroundColor(Color::srgb(0.4,0.15,0.15))
        ]
    }
}
```
`commands.spawn_scene(bsn!{...})` / `.add_systems(Startup, scene.spawn())`。

## 【原理】
场景本质是「用反射记录的一组实体/组件/资源」。序列化时按类型路径 + 字段值写 RON；反序列化用 `AppTypeRegistry` 的 type data 把动态值 `apply`/`FromReflect` 成具体类型并 spawn。`DynamicWorldRoot` 是资产句柄组件，加载完成后实例化，故能随文件热重载更新。BSN 宏在编译期把语法糖展开成普通场景构造调用，`--` 分隔兄弟节点，`@func()` 引用子场景，`Children [...]` 声明层级。

## 【最佳实践】
- 参与序列化的自定义组件/资源必须 `#[derive(Reflect)] #[reflect(Component)]`/`#[reflect(Resource)]` 并注册。
- 运行期字段用 `#[reflect(skip_serializing)]` 跳过，并在 `FromWorld` 里初始化。
- 写文件必须异步（`IoTaskPool`），WASM 无文件系统需 `#[cfg(not(target_arch="wasm32"))]`。

## 相关章节
- 反射基础：[ECS 核心 · Reflection](02-ecs-core.md#26-reflection反射)
- 资产热重载：[资产、异步与音频](09-assets-async-audio.md)
