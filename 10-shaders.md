# 10. 着色器

> [← 回到目录](README.md)
>
> ⚠️ **版本警示**：本仓库着色器用 **WESL**（`.wesl`，带 `import` 模块系统），不再用传统 `.wgsl`（`assets/shaders/` 下无 `.wgsl`）。

## 10.1 WESL 基础与 import

### 【方案】
```wgsl
// assets/shaders/custom_material.wesl
import bevy_pbr::render::forward_io::VertexOutput;
import super::custom_material_import::COLOR_MULTIPLIER;
@group(constants::MATERIAL_BIND_GROUP) @binding(0) var<uniform> material_color: vec4<f32>;
@group(constants::MATERIAL_BIND_GROUP) @binding(1) var material_color_texture: texture_2d<f32>;
@fragment
fn fragment(mesh: VertexOutput) -> @location(0) vec4<f32> {
    return material_color * textureSample(material_color_texture, mesh.uv) * COLOR_MULTIPLIER;
}
```
内置全局量：`import bevy_pbr::render::{mesh_view_bindings::globals, ..}`，用 `globals.time`。

### 【原理】
`crates/bevy_shader/src/shader.rs` 的 `scan_wesl_imports` 解析依赖按需编译。`@group(#{MATERIAL_BIND_GROUP})` 由材质系统注入组号。`globals` 是 view/世界级 uniform。

### 【最佳实践】
import 路径：`super::` 指同 assets 目录，`bevy_pbr::` 指引擎内建。时间从 `globals` 取，别自己传 uniform。

---

## 10.2 自定义材质（AsBindGroup + Material）

### 【方案】
```rust
#[derive(Asset, TypePath, AsBindGroup, Debug, Clone)]
struct CustomMaterial {
    #[uniform(0)] color: LinearRgba,
    #[texture(1)] #[sampler(2)] color_texture: Option<Handle<Image>>,
    alpha_mode: AlphaMode,
}
impl Material for CustomMaterial {
    fn fragment_shader() -> ShaderRef { SHADER_ASSET_PATH.into() }
    fn alpha_mode(&self) -> AlphaMode { self.alpha_mode }
}
app.add_plugins(MaterialPlugin::<CustomMaterial>::default());
```
变体：2D 用 `Material2d`+`Material2dPlugin`+`AlphaMode2d`；精灵材质 `MaterialExtension2d`+`SpriteMaterialPlugin`（binding 从 20 起）；数组纹理 `#[texture(0, dimension = "2d_array")]`；`Option<Handle<Image>>` 为 None 用 fallback。

### 【原理】
`AsBindGroup` derive 生成 bind group layout 与 bind group；字段属性决定槽位与资源类型。`Material` trait 提供默认实现，只需覆盖需要的。

### 【最佳实践】
binding 编号别冲突（扩展材质从 100、精灵从 20）。WebGL2 下 uniform 结构体需 16 字节对齐。纹理用 `Option` 容忍缺失。

---

## 10.3 ExtendedMaterial / 存储缓冲 / GPU 读回 / 计算着色器

- `ExtendedMaterial<B, E>`：在 `StandardMaterial` 上加自己的 uniform/纹理（`#[uniform(100)]`），`impl MaterialExtension`；forward 可改光照后结果，deferred 只能改 `PbrInput`。
- 存储缓冲：材质里 `#[storage(0, read_only)] colors: Handle<ShaderBuffer>`；`buffers.add(ShaderBuffer::from(data))`，每帧 `buffers.get_mut(&h)` 改。
- GPU 读回：缓存加 `BufferUsages::COPY_SRC`；`commands.spawn(Readback::buffer(handle)).observe(|e: On<ReadbackComplete>| e.to_shader_type())`。
- `GpuComponentArrayBuffer`：`impl GpuComponentArrayBuffer`（`type QueryData/QueryFilter/Out`，`#[repr(C)]+Pod`）把 ECS 组件批量提取为单个供所有实例索引的 buffer，配 `MeshTag` 与 `binding_array`。
- 计算着色器：`Image::new_target_texture` + `TextureUsages::{COPY_DST, STORAGE_BINDING, TEXTURE_BINDING}`；`PipelineCache::queue_compute_pipeline`；渲染图节点里 `pass.dispatch_workgroups(size.x/8, size.y/8, 1)`（`compute_shader_game_of_life.rs`）。双缓冲 ping-pong。

### 【原理】
渲染世界：`ExtractResource`/`@extract_app(RenderApp)` 把主世界资源克隆进渲染世界。管线编译是异步的，用 `CachedPipelineState` 检测就绪。

### 【最佳实践】
读回的缓冲/纹理必须显式加 `COPY_SRC`。CPU→GPU 结构体必须 `#[repr(C)]+Pod+Zeroable` 且 16 字节对齐。`dispatch_workgroups` 参数是**工作组数量**（尺寸/工作组长）。`GpuComponentArrayBuffer` 的 QueryFilter 用 `Changed<T>` 减少上传。

---

## 10.4 渲染图节点 / Prepass / 管线常量 / Instancing / shader_defs / Bindless

- 渲染图：`render_app.add_systems(RenderGraph, my_node.before(camera_driver))`，`ctx.command_encoder().begin_compute_pass(..)`。
- Prepass：相机加 `DepthPrepass`/`NormalPrepass`/`MotionVectorPrepass`；材质可 `enable_prepass()->false` 或自定义 `prepass_fragment_shader`。
- 管线常量（编译期变体，替代运行期 uniform）：`descriptor.fragment.constants.push(("LEVELS".into(), key.levels as f64))`。
- 自动实例化：同 mesh+material 自动合批，`MeshTag` 传索引。
- shader_defs：`specialize` 里 `descriptor.fragment.shader_defs.push("IS_RED".into())` + `#[bind_group_data(CustomMaterialKey)]`。
- Bindless：`#[uniform(0, BindlessMaterialUniform, binding_array(10))]` + `#[bindless(limit(4))]`。

### 【原理】
渲染图由带标签节点组成，`RenderGraph` schedule 里用 `.before(camera_driver)` 排序。管线常量在管线创建时替换进 shader 源；`shader_defs` 是编译期 `#ifdef` 开关；bindless 把多份材质打包进 binding array 用索引间接寻址。

### 【最佳实践】
自动实例化要求所有实例**共享同一 mesh + material handle**。`specialize` 的 key 要小而可哈希。后处理/prepass 显示用 `AlphaMode::Blend` 且 `enable_prepass()==false`。

---

## 相关章节
- GPU 实例化概览：[3D 渲染 · 实例化](06-3d-rendering.md)
- 自定义 UI shader：[UI 与文本](07-ui-and-text.md)
