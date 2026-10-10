# 04. 变换、数学类型与 Gizmos

> [← 回到目录](README.md)

## 4.1 Transform / GlobalTransform（层级传播）

### 【问题】
局部变换和世界变换区别？改了父实体，子实体世界位置为何自动跟着变？

### 【方案】
`Transform`（局部 `translation/rotation: Quat/scale`）+ `GlobalTransform`（只读世界变换，自动计算）。链式构造 `Transform::from_translation(...).with_rotation(...)`、`transform.forward()`、`looking_at(target, up)`。父子用 `ChildOf`/`Children`（`children![...]`）。对齐：`Transform::IDENTITY.aligned_by(Vec3::NEG_Z, *dir1, Vec3::X, *dir2)`（参数是 `Dir3`）。示例：`examples/transforms/{transform,translation,scale,3d_rotation,align}.rs`。

### 【原理】
传播管线（`crates/bevy_transform/src/plugins.rs` 的 `TransformSystems::Propagate`，在 `PostUpdate`）：
1. `sync_simple_transforms`：无父无子 → `GlobalTransform = Transform`；
2. `propagate_parent_transforms`：按层级根→叶，`global = parent_global * local`，可并行；
3. `mark_dirty_trees` 做增量。
`GlobalTransform` 内部是 `Affine3A`。渲染/物理读 `GlobalTransform`，逻辑改 `Transform`。

### 【最佳实践】
逻辑改 `Transform`，读世界位置用 `GlobalTransform`（或 `TransformHelper`）。要相机跟随/子随父就建父子关系。读相机/父级 `GlobalTransform` 的系统应排在 `TransformSystems::Propagate` 之后。旋转一律用 `Quat` 避免万向节锁。

---

## 4.2 移动 / 旋转（帧率无关）

### 【问题】
如何按时间平滑移动/旋转？如何让物体「面向」目标？

### 【方案】
```rust
// examples/2d/move_sprite.rs —— 位置更新乘 delta
transform.translation.x += 150. * time.delta_secs();
// 朝向（即时）
let dir = Quat::from_rotation_arc(Vec3::Y, to_target.extend(0.));  // rotation.rs
// 朝向（限速）：点积 + acos 求夹角，clamp 后 rotate_z(min(step, max_angle))
```
旋转角度用量纲化写法：`speed * TAU * delta_secs()` = 「每秒转 speed 圈」（`examples/transforms/3d_rotation.rs`）。

### 【原理】
`delta_secs()` 是上一帧时间，乘速度保证帧率无关。朝向数学：单位向量点积 ∈ [-1,1]，`acos` 得夹角（需 clamp 防浮点越界返回 NaN），右向量点积的符号决定顺/逆时针。

### 【最佳实践】
位置/旋转更新永远乘 `delta_secs()`。求夹角前 `clamp(-1,1)` 再 `acos`（源码专门警告）。归一化方向向量避免对角移动更快。确定性物理用 `FixedUpdate`+`Time<Fixed>`。

---

## 4.3 数学类型与曲线

### 【问题】
有哪些向量/旋转/方向/仿射类型？如何构造与采样曲线？包围体检测？

### 【方案】
- 向量/旋转：`Vec2/Vec3`、`Quat`、`Rot2`、`Mat3/Mat4`、`Affine3A`、`Isometry2d/Isometry3d`（平移+旋转、无缩放）。
- 方向类型 `Dir2/Dir3`（构造后保证单位长度，`crates/bevy_math/src/direction.rs`）。
- 曲线：`CubicHermite::new(points, tangents)`、`CubicCardinalSpline::new_catmull_rom(points)`、`CubicBSpline`，`.to_curve()` 得 `CubicCurve`，`curve.iter_positions(resolution)` 采样（`examples/math/cubic_splines.rs`）。
- 包围体：`Bounded2d`（`aabb_2d(isometry)`、`bounding_circle`、`cast_ray`）（`examples/math/bounding_2d.rs`）。
- 随机采样：`shape.sample_interior(rng)` / `sample_boundary(rng)`（`examples/math/random_sampling.rs`）。

### 【原理】
`Dir2/Dir3` 是「保证归一化」的新类型，避免反复 normalize 与浮点漂移。`Isometry` 表示刚体变换。`Curve` trait（`crates/bevy_curve/`）统一曲线接口（`position(t)`/`velocity`/分段）。

### 【最佳实践】
表示方向优先 `Dir2/Dir3`。只做「旋转+平移」用 `Isometry`。画曲线按长度调分辨率（`100 * segments().len()`），采样点需 `extend(0.0)` 才能喂 3D `linestrip`。

---

## 4.4 Gizmos（即时模式调试绘制）

### 【问题】
调试时想画射线/AABB/路径/曲线而不创建正式资产；按类别控制样式/显隐；性能要好。

### 【方案】
```rust
// examples/gizmos/2d_gizmos.rs
gizmos.line_2d(a, b, RED);
gizmos.circle_2d(Isometry2d::IDENTITY, 300., NAVY).resolution(64);
gizmos.grid_2d(Isometry2d::IDENTITY, UVec2::new(16, 9), Vec2::splat(80.), gray).outer_edges();
```
自定义分组：`#[derive(Default, Reflect, GizmoConfigGroup)]` + `app.init_gizmo_group::<G>()`，系统参数 `Gizmos<G>`。配置用 `ResMut<GizmoConfigStore>`（`config.line.width/style/joints`、`config.enabled`）。海量静态线用 `GizmoAsset` + `Gizmo` 组件（`examples/gizmos/3d_gizmos.rs`）。

### 【原理】
`Gizmos` 是每帧重建的即时缓冲，帧末上传，随用随弃，**不适合每帧绘制海量静态几何**；那种情况用 `GizmoAsset`（一次性上传、长期存在）。分组让不同类别有独立线宽/样式/开关。

### 【最佳实践】
频繁更新、线少 → `Gizmos` 系统参数；静态、线多 → `GizmoAsset` + `Gizmo` 组件。大圆/弧手动 `.resolution(...)`。用自定义 `GizmoConfigGroup` 隔离调试类别。

---

## 相关章节
- 2D 坐标转换：[2D 渲染 · 相机与坐标](05-2d-rendering.md)
- 3D 相机投影：[3D 渲染 · 相机](06-3d-rendering.md)
- 用 gizmo 调试可见性：[工具链与调试](11-tooling-and-debugging.md)
