# 05. 2D 渲染

> [← 回到目录](README.md)

## 5.1 Sprite 与 SpriteSheet / TextureAtlas

### 【问题】
2D 显示图/图集；按格子切片、按帧播放；放大时 9 宫格拉伸/平铺；大图集缩放渗色。

### 【方案】
```rust
// examples/2d/sprite_sheet.rs
let layout = TextureAtlasLayout::from_grid(UVec2::splat(24), 7, 1, None, None);
commands.spawn((
    Sprite::from_atlas_image(tex, TextureAtlas { layout: layouts.add(layout), index: first }),
    Transform::from_scale(Vec3::splat(6.0)),
    AnimationTimer(Timer::from_seconds(0.1, TimerMode::Repeating)),
));
```
- 9 宫格：`SpriteImageMode::Sliced(TextureSlicer { border: BorderRect::all(x), center_scale_mode: SliceScaleMode::Stretch|Tile{..}, .. })`（`sprite_slice.rs`）。
- 平铺：`SpriteImageMode::Tiled { tile_x, tile_y, stretch_value }`（`sprite_tile.rs`）。
- 翻转：`Sprite { flip_x, flip_y, .. }`。图集生成：`TextureAtlasBuilder::default().padding(..).build()`（`texture_atlas.rs`）。

### 【原理】
`Camera2d` 触发 2D 管线；`Sprite` 转成带 UV 的四边形。`TextureAtlasLayout` 是资源（记录每子图 `URect`）；`TextureAtlas { layout, index }` 指向某块。`Sliced` 把图切 9 块四角不变形；`Tiled` 重复 UV。`TextureAtlasBuilder` 的 `padding` 在子图间留空避免线性采样把邻图颜色插进来；`ImageSampler::nearest()` 彻底消除渗色。精灵按 `GlobalTransform` 的 z 排序。

### 【最佳实践】
像素风用 `ImagePlugin::default_nearest()`。放大画面用 `custom_size`/`scale` 区别对待（`Transform.scale` 会整体放大纹理导致像素化）。图集用 `padding` 或 `nearest` 解决渗色。

---

## 5.2 2D 网格与自定义绘制

### 【问题】
需要矢量风格/精确几何/可拾取/自定义着色器时，如何生成 `Mesh`？CPU 逐像素画图？

### 【方案】
```rust
// examples/2d/mesh2d.rs
commands.spawn((Mesh2d(meshes.add(Rectangle::default())),
                MeshMaterial2d(materials.add(Color::from(PURPLE))),
                Transform::default().with_scale(Vec3::splat(128.))));
```
手写网格（`mesh2d_manual.rs`）：`Mesh::new(PrimitiveTopology::TriangleList, RenderAssetUsages::RENDER_WORLD)` + `insert_attribute(MeshVertexAttribute::new("Vertex_Color", 1, VertexFormat::Uint32), ...)` + `insert_indices`，三角形须 **CCW 绕序**。CPU 绘制：`Image::new_fill(...)` + `set_color_at/get_color_at`（`cpu_draw.rs`，放 `FixedUpdate`）。

### 【原理】
`Mesh2d`/`MeshMaterial2d` 是新类型包装，避免与 3D 句柄歧义。2D 与 3D 共用几何但用独立 `Mesh2dPipeline`。`mesh2d_manual.rs` 展示「中层级 API」：自定义 `RenderCommand`、`SpecializedRenderPipeline`、顶点缓冲布局、bind group。`RENDER_WORLD` 表示上传 GPU 后丢弃 CPU 端数据。

### 【最佳实践】
简单需求用高层 `Mesh2d`，只在需自定义属性/着色器时才落到底层。注意逆时针绕序否则被背面剔除。大量逐像素写入用 `MAIN_WORLD | RENDER_WORLD` 并配固定时间步。

---

## 5.3 2D 形状、Bloom 与 Text2D

### 【问题】
快速搭形状场景；让亮色发光；世界空间文字。

### 【方案】
```rust
// examples/2d/bloom_2d.rs
commands.spawn((Camera2d, Hdr, Tonemapping::TonyMcMapface, Bloom::default(), DebandDither::Enabled));
// 亮色物（分量 > 1 才发光）：
MeshMaterial2d(materials.add(Color::srgb(7.5, 0.0, 7.5)))
```
Text2D（`examples/2d/text2d.rs`）：`Text2d::new("...")` + `TextFont` + `TextLayout::new(Justify, LineBreak)` + `TextBounds::from(size)` + `Text2dShadow` + 子实体 `TextSpan`。

### 【原理】
Bloom 是后处理：阈值/预滤波 → 多级模糊 → 按 `composite_mode` 合成。HDR（RGB>1）在色调映射前保留能量，才能越过阈值发光。Text2d 是世界空间文本实体，随 `Transform` 变换。

### 【最佳实践】
想发光必须给分量 > 1 的颜色。缩放文字改 `font_size` 而非 `Transform.scale`（后者放大已栅格化字形导致像素化）。

---

## 5.4 2D 相机投影与坐标转换

### 【问题】
2D 相机如何正交/缩放？如何让视口高度随窗口一致？屏幕坐标↔世界坐标？

### 【方案】
```rust
// examples/camera/projection_zoom.rs —— 缩放改 scale，不移动相机
Projection::from(OrthographicProjection {
    scaling_mode: ScalingMode::FixedVertical { viewport_height: 5. },
    scale: 1., ..OrthographicProjection::default_3d()
})
let multiplicative_zoom = 1. + (-scroll * speed);
orthographic.scale = (orthographic.scale * multiplicative_zoom).clamp(min, max);
```
屏幕→世界：`camera.viewport_to_world_2d(camera_transform, cursor)`；反向 `world_to_viewport`（`examples/2d/2d_viewport_to_world.rs`）。注意绘制系统排在 `PostUpdate` 且 `.after(TransformSystems::Propagate)`。

### 【原理】
正交投影 `scale` 是「世界单位→屏幕」因子（`scale=1` 时 1 世界单位=1 逻辑像素）。`ScalingMode`（`crates/bevy_camera/src/projection.rs`）：`WindowSize`（默认，世界高度=窗口像素高）、`FixedVertical{viewport_height}`（世界高度固定）、`FixedHorizontal`、`AutoMin/Max`。

### 【最佳实践】
缩放改 `scale`、用对数式（`scale *= (1+delta)`）而非线性加减。读取相机 `GlobalTransform` 的系统排在传播之后。用「往返转换」自测坐标正确性。

---

## 相关章节
- Transform 传播：[变换、数学与 Gizmos](04-transform-math-gizmos.md)
- 3D 相机：[3D 渲染](06-3d-rendering.md)
- UI（独立坐标系）：[UI 与文本](07-ui-and-text.md)
