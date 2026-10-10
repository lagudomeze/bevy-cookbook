# 09. 资产 / 异步任务 / 音频

> [← 回到目录](README.md)

## 9.1 AssetServer、Handle 与引用计数

### 【问题】
一句路径取资产又不复制数据？Handle 何时让资产被释放？

### 【方案】
```rust
// examples/asset/asset_loading.rs
let cube = asset_server.load(GltfAssetLabel::Primitive { mesh: 0, primitive: 0 }
                                .from_asset("models/cube/cube.gltf"));
let mat = materials.add(StandardMaterial { base_color: Color::srgb(0.8,0.7,0.6), ..default() });
commands.spawn((Mesh3d(cube), MeshMaterial3d(mat.clone())));
```
`alter_sprite.rs` 关键差异：改 **Handle**（`sprite.image = ...`）只换这一个实体的引用；改 **Asset**（`images.get_mut(&handle)`）影响所有引用者。

### 【原理】
`crates/bevy_asset/src/handle.rs`：`Handle<A>` 是 `Strong(Arc<StrongHandle>)` / `Uuid` 两变体（**本仓库已无 `Weak`**）。`Strong` 的 `Drop` 通知 AssetServer 递减引用计数，归零则释放（`AssetEvent::Unused/Removed`）。`Uuid` 是稳定 ID、不保活。数据存于 `Assets<A>` 资源。

### 【最佳实践】
`load()` 的强 handle 必须保存到组件/资源，否则无人持有即释放。`load_folder()` 返回的 `Handle<LoadedFolder>` 也要保存。运行时才要改的网格/贴图必须用 `RenderAssetUsages::MAIN_WORLD`。

---

## 9.2 加载状态、子资产、热重载、自定义 loader、处理管线、来源

- 加载就绪判定：`asset_server.load_state(id)`（NotLoaded/Loading/Loaded/Failed）；或 `AssetBarrier` + 运行条件（`multi_asset_sync.rs`）。
- 子资产：`AssetPath = <source>://<path>#<label>`；glTF 用 `GltfAssetLabel`；loader 侧 `load_context.add_labeled_asset(label, asset)`。
- 热重载：需 `file_watcher` feature；文件变更后重走 loader，发 `AssetEvent::Modified`，持 handle 的组件自动看到新数据。
- 自定义 loader：`impl AssetLoader`，`async fn load(&self, reader, settings, ctx)`，`fn extensions()`；`app.init_asset::<T>().init_asset_loader::<L>()`。
- 处理管线：`AssetMode::Processed` + `register_asset_processor::<LoadTransformAndSave<L, T, S>>(...)`，构建期生成 mipmap/压缩纹理/格式转换。
- 来源：`app.register_asset_source("name", AssetSourceBuilder::..)`（须在 `AssetPlugin` 前）；`embedded_asset!` 编进二进制；`https://` 来源。
- 事件与生成：`AssetEvent<A>`（Added/Modified/Removed/Unused/LoadedWithDependencies）配 `run_if(on_message::<AssetEvent<T>>)`；`materials.add()`/`asset_server.add_async()`/`meshes.reserve_handle()`。

### 【原理】
`AssetLoader::load` 是 async，`Reader` 异步读字节，`LoadContext` 可解析相对路径、发起子加载。`AssetSource` = `AssetReader`（只懂字节）+ 路径根；`AssetPath` 的 `<source>://` 选来源。

### 【最佳实践】
别在 Startup 假设资产就绪。读写文件放 `IoTaskPool`。`with_settings` 不能改变已加载资产的设置，需不同参数时复制文件或用 `.meta`。

---

## 9.3 异步任务（TaskPool 家族）

### 【问题】
重计算/IO 不能卡主线程，三个池该用哪个？任务结果怎么拿？

### 【方案】
```rust
// examples/async_tasks/async_compute.rs
let task = AsyncComputeTaskPool::get().spawn(async move {
    Delay::new(duration).await;
    let mut cq = CommandQueue::default();
    cq.push(move |world: &mut World| { /* 在主线程安全地改 World */ });
    cq
});
commands.entity(entity).insert(ComputeTransform(task));
// 主线程轮询
if let Some(mut cq) = check_ready(&mut task.0) { commands.append(&mut cq); }
```

### 【原理】
`crates/bevy_tasks/src/usages.rs` 定义三个全局单例：`ComputeTaskPool`（**本帧内必须完成**的密集工作）、`AsyncComputeTaskPool`（**可跨多帧**）、`IoTaskPool`（IO 密集、多处于未唤醒）。Web 端在主线程 tick（WASM 无多线程）。`spawn` 返回 `Task<T>`；`check_ready` 非阻塞轮询。

### 【最佳实践】
会阻塞的别放 `ComputeTaskPool`。**不要 `block_on`**，用 `check_ready`。结果涉及 ECS 变更用 `CommandQueue`/`Commands` 延迟应用。Channel 模式（`async_channel_pattern.rs`）适合「完成即上报」；有界通道做背压（`external_source_external_thread.rs`）。

---

## 9.4 音频

### 【方案】
```rust
// examples/audio/audio.rs
commands.spawn(AudioPlayer::new(asset_server.load("sounds/Windless Slopes.ogg")));
// 一次性音效播完销毁
commands.spawn((AudioPlayer::new(sfx.clone()), PlaybackSettings::DESPAWN));
// 运行期控制（audio_control.rs）
sink.toggle_playback(); sink.set_volume(..); sink.set_speed(..); sink.position();
// 空间音频（spatial_audio_3d.rs）
PlaybackSettings::LOOP.with_spatial(true) + SpatialListener::new(gap)
// 自定义音源（decodable.rs）
impl Source for SineDecoder { fn channels()..; fn sample_rate()..; }
impl Decodable for SineAudio { type Decoder = SineDecoder; .. }
app.add_audio_source::<SineAudio>();
```

### 【原理】
`AudioSource`（内部 rodio `Decoder`）是默认资产；`AudioPlayer<Source>` 是组件，spawn 后生成播放器/sink。`AudioSink` 封装 rodio Sink（`set_volume/set_speed/toggle_playback/position`）。`SpatialListener` 用两子实体表示左右耳。`Decodable`：`type Decoder: Source + Iterator<Item=Sample>`。

### 【最佳实践】
一次性音效用 `PlaybackSettings::DESPAWN` 自动清理。预加载 handle 到资源。新曲先 `Volume::SILENT` 启动再淡入避免爆音。sink 组件在音频真正开始后才存在，用 `single()` 的 Ok/Err 处理未就绪。

---

## 相关章节
- 场景/世界序列化：[场景与序列化](03-scene-and-serialization.md)
- 异步任务的 channel 模式在外部数据源接入：[ECS 核心 · 协程](02-ecs-core.md#24-协程--一次性系统--回调)
