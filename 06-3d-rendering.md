# 06. 3D 渲染

> [← 回到目录](README.md)

## 6.1 3D 相机与投影

### 【问题】
默认透视相机的正交切换、视野控制。

### 【方案】
```rust
// examples/3d/orthographic.rs
Projection::from(OrthographicProjection {
    scaling_mode: ScalingMode::FixedVertical { viewport_height: 6.0 },
    ..OrthographicProjection::default_3d()
})
// 透视：PerspectiveProjection { fov, near, far, ..default() }
```
运行期切换用 `Projection::Perspective(_)` / `Projection::Orthographic(_)`。

### 【原理】
`Camera3d` 在 `ExtractSchedule` 打包为 `ExtractedCamera` 送入 RenderApp，`Projection` 换算成 `ViewUniform.projection_matrix`。正交是仿射变换（无透视除法）。

### 【最佳实践】
多相机务必给 `Camera.order` 定绘制顺序。正交相机无透视收缩，AABB/拾取仍用世界坐标。

---

## 6.2 PBR 材质（StandardMaterial）

### 【问题】
金属度/粗糙度怎么调？贴图怎么加？清漆/各向异性/透射？

### 【方案】
```rust
// examples/3d/texture.rs
MeshMaterial3d(materials.add(StandardMaterial {
    base_color: Color::srgb(0.8, 0.2, 0.2),
    base_color_texture: Some(asset_server.load("textures/uv_checker_bw.png")),
    alpha_mode: AlphaMode::Blend, ..default()
}))
```
高级字段：`clearcoat*`（`clearcoat.rs`）、`anisotropy_*`（`anisotropy.rs`）、`specular_transmission`/`thickness`/`ior`（`transmission.rs`）、`depth_map`+`parallax_*`（`parallax_mapping.rs`）。

### 【原理】
标准 Cook-Torrance 微表面模型：`base_color` 是漫反射反照率，`metallic` 插值介电/金属，GGX 法线分布由 `perceptual_roughness`（感知粗糙度，内部平方）控制。`Material` trait 经 `specialize()` 把材质字段编进 `MeshPipelineKey` 位标志，每种组合编译一个管线变体。

### 【最佳实践】
- 要「自动实例化」，同一 `Handle<Mesh>` + 同一 `Handle<Material>` 是前提。
- `alpha_mode` 必须匹配需求：不透明却设 `Blend` 会进透明队列、禁预通道、破坏排序。
- 色彩空间：base color 贴图是 sRGB，法线/粗糙度/金属是线性。
- 用法线贴图先 `mesh.with_generated_tangents()`。
- 扩展优先用 `ExtendedMaterial<StandardMaterial, MyExt>`（绑定槽从 100 起），别从零写材质。

---

## 6.3 光源

### 【问题】
平行光/点光/聚光怎么加？阴影、级联、环境光、环境贴图、光探针？

### 【方案】
```rust
// examples/3d/lighting.rs
commands.spawn((SpotLight {
    intensity: 4_000_000.0, inner_angle: 0.6, outer_angle: 0.8,
    shadow_maps_enabled: true, ..default()
}, Transform::from_xyz(4., 8., 4.).looking_at(Vec3::ZERO, Vec3::Y)));
```
级联阴影：`CascadeShadowConfigBuilder { first_cascade_far_bound, maximum_distance, .. }.build()`。全局环境光：`GlobalAmbientLight { brightness }`。环境贴图：`EnvironmentMapLight { diffuse_map, specular_map, intensity, rotation }`。光探针：`LightProbe`+`EnvironmentMapLight`+`ParallaxCorrection`（`reflection_probes.rs`）。

### 【原理】
前向（Forward+）渲染中每像素对所有可见光源求和；`Clusters` 把视锥切成 3D 网格剔除后写光源索引，支撑大量光源。阴影从光源视角渲深度（方向光单张 2D+级联，点光立方体贴图），主 pass 用 PCF。`NotShadowCaster`/`NotShadowReceiver` 在实体侧开关。

### 【最佳实践】
点光阴影是立方体贴图，开销是方向光 6 倍，能不用就不用。级联调 `first_cascade_far_bound`/`maximum_distance` 权衡近清远覆。环境贴图强度与 `GlobalAmbientLight` 会叠加，别重复提亮。

---

## 6.4 3D 几何与网格

### 【方案】
内置图元：`Cuboid`/`Sphere`/`Capsule3d`/`Torus`/`Cylinder`/`Cone`/`ConicalFrustum`/`Tetrahedron`/`Plane3d`/`Rectangle`/`Circle`/`Extrusion`/`Ring`（`examples/3d/3d_shapes.rs`）。手写网格 `examples/3d/generate_custom_mesh.rs`（`insert_attribute(POSITION/NORMAL/UV_0)` + `with_inserted_indices`）。

### 【原理】
`Mesh` 是 CPU 侧顶点属性 + 索引，`insert_attribute` 按 `MeshVertexBufferLayout` 铺进 GPU 顶点缓冲；`RenderAssetUsages` 决定保留在 main/render world 哪一侧。`MeshAllocator` 把顶点/索引打包进共享大缓冲以实现合批。

### 【最佳实践】
只渲染不再改的网格设 `RENDER_WORLD` 省内存；要运行期改则须含 `MAIN_WORLD`。法线要归一化。频繁改顶点会破坏打包并触发重上传。

---

## 6.5 线框、雾、大气

### 【方案】
线框：`WireframePlugin` + 逐实体 `Wireframe`/`WireframeColor`/`WireframeLineWidth`，`NoWireframe` 排除（`wireframe.rs`）。距离雾：相机上 `DistanceFog { color, falloff: FogFalloff::ExponentialSquared{density}, .. }`。体积雾：`VolumetricFog`（相机）+`VolumetricLight`（光源）+`FogVolume`（实体），**必须有阴影**。大气：`Atmosphere::earth()/mars()` + `AtmosphereSettings` + `AtmosphereMode::{LookupTexture, Raymarched}`。天空盒：`Skybox { image, brightness }` + 立方体贴图视图。

### 【原理】
距离雾是片元阶段按距离/高度混合雾色（纯后处理近似）。大气是物理散射模型：`LookupTexture` 预计算 LUT（快），`Raymarched` 逐像素求积（准但贵）。天空盒画在最远深度，`Skybox` 与 `EnvironmentMapLight` 常成对用保证光照一致。

### 【最佳实践】
线框依赖多边形填充模式，移动端/WebGL 常不可用。体积雾很吃性能，限定范围。大气示例强制 `DefaultOpaqueRendererMethod::deferred()` 且 `GlobalAmbientLight::NONE`。距离雾色要与天空底部色一致，否则地平线割裂。

---

## 6.6 色调映射与颜色分级

### 【方案】
```rust
// examples/3d/tonemapping.rs
commands.spawn((Camera3d::default(), Hdr,
    Tonemapping::TonyMcMapface,   // 或 AgX/AcesFitted/Reinhard/.../KhronosPbrNeutral
    ColorGrading { global: ColorGradingGlobal { exposure, temperature, tint, hue, post_saturation }, ..default() }));
```
自动曝光：`AutoExposurePlugin` + `AutoExposure { metering_mask, compensation_curve }`（`auto_exposure.rs`）。

### 【原理】
后处理链：HDR 线性渲染 → AutoExposure（全屏平均亮度反馈曝光）→ Tonemapping（HDR→[0,1]）→ ColorGrading（lift/gamma/gain/对比度/色温）→ 输出。AgX/TonyMcMapface 保留高光色彩与暗部细节。

### 【最佳实践】
用 HDR+Bloom 时**不要**用 `Tonemapping::None/Linear`，否则高光死白。先定 Tonemapping 再 ColorGrading，顺序不可颠倒。

---

## 6.7 辉光 / 自发光

### 【方案】
相机 `Hdr`+`Bloom::NATURAL`+`Tonemapping::TonyMcMapface`；材质 `emissive: LinearRgba::rgb(5.0, 0.4, 0.4)`（可 >1.0）。**必须有 `Hdr`**，否则 emissive 被 clamp 到 1.0。

### 【原理】
Bloom = 阈值提取亮部 → 多级降采样高斯模糊 → 升采样合成。`EnergyConserving` 从原图扣能量避免失真。

### 【最佳实践】
自发光不等于光源（不照亮周围），要照亮需额外放光源。

---

## 6.8 分屏与视口

### 【方案】
多相机 + `Camera.order` + `Viewport { physical_position, physical_size, .. }`，窗口 resize 时用 `MessageReader<WindowResized>` 重算（`split_screen.rs`）。UI 用 `UiTargetCamera` 绑定相机。

### 【最佳实践】
分屏时每相机设 `Camera.clear_color`（一个 `Clear`，其余 `None`/指定色）。resize 必须重算 `physical_size`。

---

## 6.9 透明与 Alpha 模式

### 【问题】
半透明排序错误/闪烁？OIT？

### 【方案】
`AlphaMode` 四档：`Opaque`、`Mask(f32)`、`Blend`、`AlphaToCoverage`（`transparency_3d.rs`）；blend 模式 `Premultiplied`/`Add`/`Multiply`（`blend_modes.rs`）。OIT：相机加 `OrderIndependentTransparencySettings` + `DepthPrepass` + `Msaa::Off`。

### 【原理】
`Opaque` 走不透明队列、写深度、可预通道、可实例化。`Mask` 片元 discard（树叶），不透明管线但有 alpha 测试。`Blend` 走透明队列、按距相机远→近排序、关深度写入，穿插时会排序错误——OIT 用深度预通道 + 逐像素链表/加权求和消除排序依赖，代价是需 `Msaa::Off`。

### 【最佳实践】
能 `Mask` 就不 `Blend`；能 `Opaque` 就别透明。大量树叶务必 `Mask`。OIT 不支持 MSAA。

---

## 6.10 GPU 实例化与自定义材质

### 【方案】
```rust
// examples/shader/automatic_instancing.rs —— 同 mesh+material 自动合批，无需代码
commands.spawn((Mesh3d(mesh.clone()), MeshMaterial3d(mat.clone()), MeshTag::new(index), Transform::..));
// 自定义材质（shader_material.rs）
#[derive(Asset, TypePath, AsBindGroup, Clone)]
struct CustomMaterial { #[uniform(0)] color: LinearRgba,
    #[texture(1)] #[sampler(2)] color_texture: Option<Handle<Image>>, alpha_mode: AlphaMode }
impl Material for CustomMaterial { fn fragment_shader() -> ShaderRef { SHADER_ASSET_PATH.into() } }
app.add_plugins(MaterialPlugin::<CustomMaterial>::default());
```
完全自定义实例化见 `examples/shader_advanced/custom_shader_instancing.rs`（配 `NoFrustumCulling` + 相机 `NoIndirectDrawing`）；最底层手写材质见 `manual_material.rs`。详见 [着色器](10-shaders.md)。

### 【原理】
`BatchingPlugin`/`gpu_preprocessing` 把相同 mesh+material 的实体分组，把每实例 `MeshUniform` 写进共享缓冲，一次 `draw_indexed` 带实例数渲染。`MeshTag` 进额外槽位。自定义实例化要 `NoFrustumCulling`，因视锥剔除基于 mesh 的 AABB，实例世界变换不在其中会误剔除。

### 【最佳实践】
先试内置自动实例化，不够再自定义。共享 handle 是自动实例化的唯一前提。顶点属性 `shader_location` 别与内置 Position/Normal/UV（0-2）冲突，实例数据从 3 起。

---

## 6.11 glTF 加载

### 【方案】
```rust
// examples/gltf/load_gltf.rs
commands.spawn((WorldAssetRoot(asset_server.load(
        GltfAssetLabel::Scene(0).from_asset("models/FlightHelmet/FlightHelmet.gltf"))),
    Transform::from_scale(Vec3::splat(8.0))));
```
运行期改材质：查询 `GltfMaterialName`/`GltfMeshName` 拿 `MeshMaterial3d<StandardMaterial>` 改；或观察者等 `On<WorldInstanceReady>` 就绪后再改（`edit_material_on_gltf.rs`）。读 extras：`GltfExtras`/`GltfSceneExtras`/`GltfMeshExtras`（`load_gltf_extras.rs`）。蒙皮：`SkinnedMesh` 组件 + 关节层级。

### 【原理】
glTF 加载器把文件解析成 `Gltf` 资产，`GltfAssetLabel::Scene(n)` 取出第 n 场景层级实例化为实体树；`WorldInstanceReady` 是实例化完成的观察者事件。命名信息存在 `GltfMeshName`/`GltfMaterialName` 组件上。材质每个 primitive 一个实例，便于逐个改。

### 【最佳实践】
用 `GltfAssetLabel::Scene(n)` 明确场景。等就绪用 `On<WorldInstanceReady>` 观察者，别在 Startup 盲查。大模型配 `DirectionalLightShadowMap { size: 4096 }`。

---

## 6.12 动画

### 【方案】
```rust
// examples/animation/animated_mesh_control.rs
let (graph, index) = AnimationGraph::from_clips([animations.named_animations["Run"].clone(),
                                                animations.named_animations["Walk"].clone()]);
let mut player = AnimationPlayer::default();
player.play(index).repeat();
commands.spawn((WorldAssetRoot(scene), AnimationGraphHandle(graphs.add(graph)), player, AnimationTransitions::new()));
```
完整动画图（混合/叠加）：`AnimationGraph::new()` + `add_clip`/`add_blend`/`add_additive_blend` + `.animgraph.ron`。骨骼遮罩：`add_clip_with_mask` + `AnimationTargetId::from_names`（`animation_masks.rs`）。动画事件：`AnimationEvent` 派生 + `AnimationClip::add_event` + 观察者（`animation_events.rs`）。

### 【原理】
`AnimationGraph` 是求值树：叶子是 `AnimationClip`（关键帧采样器），内部节点是混合/叠加/遮罩节点，每帧从根向下求值得各 `AnimationTarget` 变换。`AnimationPlayer` 驱动播放状态。图可序列化为 RON 实现数据驱动状态机。遮罩把求值限制到指定骨骼（上半身挥手、下半身走路）。

### 【最佳实践】
简单循环用 `from_clip(s)`；需状态机/混合才引入完整图。过渡用 `AnimationTransitions::play(anim, Duration)`，别手动插值权重。动画对蒙皮网格必须等场景就绪。

---

## 6.13 拾取（Picking）

### 【方案】
```rust
// examples/picking/mesh_picking.rs
app.add_plugins(MeshPickingPlugin);           // 非默认插件，必须手动加！
app.add_observer(|over: On<PointerOver>| { .. });
commands.spawn((Mesh3d(..), MeshMaterial3d(..), Pickable::default()));
```
无指针的底层射线：`MeshRayCast`/`MeshRayCastSettings`/`RayCastVisibility` + `Ray3d`（`examples/3d/mesh_ray_cast.rs`）。自定义命中信息：`HitDataExtra` + `HitData::new_with_extra`（`custom_hit_data.rs`）。

### 【原理】
`bevy_picking` 是分层事件系统：后端（pointer/mesh/sprite）把物理输入转成 `PointerHits`（指针位置+命中实体+深度+附加数据），核心层按深度排序后派发 `PointerOver/Out/Press/Release/Drag` 等观察者事件。网格后端用相机射线与 AABB/三角形求交。

### 【最佳实践】
默认不含 `MeshPickingPlugin`，忘了加会导致 3D 点击无反应。只需检测不需事件时用 `MeshRayCast` 更轻量。大量物体用 AABB 预筛。

---

## 6.14 大场景与性能

### 【方案】
`VisibilityRange { start_margin, end_margin, use_aabb }` 做 HLOD 切换（配 `NoCpuCulling`，`visibility_range.rs`）；`StaticTransformOptimizations` 让不动的实体跳过变换传播（`bevy_city.rs`）；性能对比开关 `--no-automatic-batching`、`--no-indirect-drawing`、`--no-frustum-culling`、`--no-view-occlusion-culling`、`--no-cpu-culling`；`mipmap_generator.rs` 生成 mipmap 与 BCn 压缩。

### 【原理】
大场景瓶颈常在 CPU 侧提取/剔除/变换传播与 draw call 数。对策：视锥+遮挡剔除（GPU 查询）、GPU 实例化/间接绘制、静态变换缓存、HLOD、mipmap、异步流式加载。

### 【最佳实践】
先量后调，用 `--no-*` 开关二分定位瓶颈。静态物体加 `StaticTransformOptimizations`。远景用 `VisibilityRange` HLOD。纹理必开 mipmap。

---

## 相关章节
- 着色器与自定义材质：[着色器](10-shaders.md)
- 资产加载 glTF：[资产、异步与音频](09-assets-async-audio.md)
- 压测示例：[性能与压测](12-performance-stress-tests.md)
