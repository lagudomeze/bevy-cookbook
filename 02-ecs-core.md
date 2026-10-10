# 02. ECS 核心：查询 / 命令 / 资源 / 消息 / 观察者 / 状态 / 反射

> [← 回到目录](README.md)

## 2.1 Commands vs 直接访问 World

### 【问题】
系统默认并行，为什么不能随便拿 `&mut World`？`Commands` 为什么「延迟生效」？

### 【方案】（`examples/ecs/ecs_guide.rs` 正面对比）
```rust
fn new_player_system(mut commands: Commands) {          // 排队，稍后应用
    commands.spawn((Player { .. }, Score { value: 0 }));
}
fn exclusive_player_system(world: &mut World) {         // 立即、独占、阻塞并行
    let total = world.resource_mut::<GameState>().total_players;
    world.spawn((Player { .. }, Score { value: 0 }));
}
```
延迟命令进阶见 `examples/ecs/delayed_commands.rs`（`commands.delayed().secs(d).entity(e).insert(...)`）；命令错误处理见 `examples/ecs/error_handling.rs`（`commands.queue_handled(closure, handler)`、`app.set_error_handler`）。

### 【原理】
`Commands` 是命令队列，系统运行期只写队列，到 schedule 同步点（apply deferred）才施加到 `World`。因此同系统内先 `spawn` 后 `query` 拿不到新实体。独占系统 `&mut World` **独占整个 World**，调度器无法与之并行——故除初始化/序列化外应避免。

### 【最佳实践】
绝大多数逻辑用 `Commands`；只在需「同帧读写立即可见」或反射/序列化时用独占系统。用 `commands.entity(e).remove::<T>()/despawn()` 而非手改 World。

---

## 2.2 Queries：过滤、变更检测、并行、不相交

### 【问题】
如何筛选实体（有/无某组件、刚变更/新增）？`Changed`/`Added` 到底检测什么？怎么并行遍历？如何安全地让同一组件既读又写？

### 【方案】
```rust
// examples/ecs/change_detection.rs
fn change_detection(
    changed: Query<Ref<MyComponent>, Changed<MyComponent>>,
    res: Res<MyResource>,
) { /* is_added()/is_changed()/changed_by() */ }
```
- 不相交：`Single<&mut Transform, (With<Camera>, Without<Star>)>` 配另一 `Single<&Transform, With<Star>>`（`examples/ecs/iter_combinations.rs`）。
- 并行：`query.par_iter_mut().for_each(...)`，可 `.batching_strategy(BatchingStrategy::fixed(32))`（`examples/ecs/parallel_query.rs`）。
- 组合迭代：`iter_combinations_mut()` 做 N 体两两作用（`examples/ecs/iter_combinations.rs`）。
- 自定义 QueryData 突破元组 15 上限：`#[derive(QueryData)] #[query_data(mutable)]`（`examples/ecs/custom_query_param.rs`）。
- 连续内存迭代（可 SIMD）：`query.contiguous_iter_mut().unwrap()`（`examples/ecs/contiguous_query.rs`）。
- 语义化参数：`Single<D,F>`（恰好一个）、`Populated<D,F>`（至少一个）、`Option<Single<...>>`（0 或 1）（`examples/ecs/fallible_params.rs`）。

### 【原理】
Bevy 按 **archetype**（组件组合）分块存储，过滤器在 archetype 层匹配，`With/Without` 几乎零成本。`Changed`/`Added` 不是比较值，而是比较**本系统上次运行之后的变更 tick**；在可变解引用时触发，**即使写入值没变也会标记已变更**。`par_iter` 经 `ComputeTaskPool` 分发。

### 【最佳实践】
- 避免无意义变更触发：用 `set_if_neq`（需 `PartialEq`），资源同理 `ResMut::set_if_neq`。
- `Ref<T>` 只是访问变更信息、不过滤；要过滤必须显式加 `Changed<T>`。
- `examples/ecs/parallel_query.rs` 警告：对 128 个元素的廉价运算用 `par_iter` **通常更慢**（并行有开销）。
- `Single`/`Populated` 校验失败会静默跳过系统（可 `set_error_handler` 改成报错）。

---

## 2.3 Resources、Local、Message（缓冲事件）、Observer（观察者）

### 【问题】
全局状态放哪？系统私有状态怎么写？**「事件」和「消息」有什么区别？**

> 本仓库关键概念：**缓冲型事件叫 `Message`**（`MessageReader/Writer/Mutator`），**`Event` 专指观察者事件**（`On<...>`、`EntityEvent`）。

### 【方案】
- 资源：`#[derive(Resource)]` + `Res<T>`/`ResMut<T>`；`init_resource::<T>()` 或 `insert_resource(v)`。
- 系统私有状态：`Local<T>`，首帧 `FromWorld` 初始化，之后跨帧保留（`examples/ecs/ecs_guide.rs`）。
- **Message**：`#[derive(Message)]` + `app.add_message::<T>()`；生产 `MessageWriter<T>::write(...)`；消费 `MessageReader<T>::read()`；读写同一类型需 `MessageMutator<T>`。见 `examples/ecs/message.rs`。
- **Observer**：`#[derive(Event)]`（全局）或 `#[derive(EntityEvent)]`（针对实体）。`commands.trigger(ev)` 触发，`.add_observer(sys)` / `entity.observe(sys)` 监听。见 `examples/ecs/observers.rs`。事件冒泡（子→父）：`#[entity_event(propagate, auto_propagate)]` + `On::propagate(false)` 阻断（`examples/ecs/observer_propagation.rs`）。

### 【原理】
资源是唯一全局单例，`Res/ResMut` 参与调度冲突判定。`Local` 属于系统实例（不占 World）。Message 存于 `Messages<T>` 资源，读者用每系统游标追踪已读位置，未读消息两帧后被清空（`examples/ecs/message.rs` 用 `.chain()` 保证 Writer 先于 Reader）。Observer 是挂 `Observer` 组件的实体，触发时**立即同步执行**（区别于 Message 的缓冲），开销低但要留意重入与顺序。

### 【最佳实践】
- 高频、多消费者、需延迟处理 → 用 **Message**；需「立即反应、定向实体、冒泡」→ 用 **Observer**（如 UI 点击、组件生命周期）。
- 生命周期钩子（component hooks）比 `On<Add/Remove>` 观察者开销更低，但一个组件每种钩子只能有一个，适合维护索引（`examples/ecs/component_hooks.rs`）。
- 不可变组件 `#[component(immutable)]` 一旦插入只能读/移除/替换，配合 `on_insert` 可完全捕获变更（`examples/ecs/immutable_components.rs`）。
- 陷阱：同系统内 `MessageReader` 与 `MessageWriter` 同类型不能共存 → 用 `MessageMutator`。

---

## 2.4 协程 / 一次性系统 / 回调

### 【问题】
同步 API 与异步如何衔接？如何把系统当「回调」存起来以后再跑？

### 【方案】
- 阻塞 IO 丢进 `IoTaskPool`：`IoTaskPool::get().spawn(async { ... }).detach()`。
- 一次性系统：`commands.register_system(sys)` / `world.register_system(sys)` 得 `SystemId`，随时 `commands.run_system(id)`、`world.run_system_once(sys)`（`examples/ecs/one_shot_systems.rs`）。
- 把闭包/系统存进组件形成「任意可调用逻辑」（`examples/ecs/callbacks.rs`）。

### 【原理】
`IoTaskPool`/`ComputeTaskPool` 是全局线程池，`spawn` 返回 `Task<T>`，`.detach()` 表示「后台跑完即弃」。`SystemId` 是已注册系统的轻量句柄，`run_system` 会把该系统排入命令队列，下次应用命令时运行。

### 【最佳实践】
任何阻塞 API（文件/网络/加解密）都别直接放系统里，交给 `IoTaskPool`。`run_system` 是延迟的，别在同帧依赖其副作用立即可见。

---

## 2.5 States / SubStates / ComputedStates

### 【问题】
如何用状态控制「哪些逻辑何时跑」？进出做初始化/清理？子状态、派生状态如何组织？「转化为自身（identity transition）」为什么不触发？

### 【先建立心智模型】
`State<S>` 就是一个**全局 Resource**：
- 资源**存在**且值等于 X → 当前处于状态 X
- 资源**不存在** → 当前没有这个状态

第二条（状态的「消失」）是一等公民，`SubStates`/`ComputedStates` 的价值就在于此。**注意：`States` 天生是全局单例，不是挂在某个实体上的**；要做「每个实体自己的状态」，见 [每实体状态与派生数值](14-per-entity-state-and-derived-stats.md)。

### 【方案】
基础三件套（`examples/state/states.rs`）：
```rust
#[derive(States, Default, Debug, Clone, Copy, Eq, PartialEq, Hash)]
enum AppState { #[default] Menu, InGame }

app.init_state::<AppState>()
   .add_systems(OnEnter(AppState::Menu), setup_menu)               // 进入时跑一次
   .add_systems(Update, hover_style.run_if(in_state(AppState::Menu))) // 每帧判断
   .add_systems(OnExit(AppState::Menu), cleanup_menu);             // 离开时跑一次
// 切换：ResMut<NextState<AppState>>::set(...)
```
**核心区分**：`OnEnter/OnExit(值)` 是 **Schedule**（切换那一刻才跑，做 setup/teardown）；`Update + .run_if(in_state(值))` 是**每帧条件**（系统一直在，条件假就跳过）。

**为什么要 SubStates / ComputedStates**：真实游戏状态是多个正交维度（菜单/游戏中 × 暂停/运行 × 涡轮/普通）。全塞进一个枚举会组合爆炸：
```rust
enum AppState { Menu, InGame { paused: bool, turbo: bool } }   // 光 InGame 就有 4 种组合
```
解法：**每个正交维度拆成一个独立状态**，但受父状态约束存在性。

- **子状态（可手动改）**（`examples/state/sub_states.rs`）：
```rust
#[derive(SubStates, Default, ...)]
#[source(AppState = AppState::InGame)]    // 仅当 AppState==InGame 时存在
enum IsPaused { #[default] Running, Paused }
app.add_sub_state::<IsPaused>();
// 存在期间可手动切：next_state.set(IsPaused::Paused)
```
`should_exist` 返回 `Some` 表示「应该在」（首次据此初始化；**资源已存在时返回值被忽略**，故手动改动不被覆盖）；返回 `None` 则移除资源。`SubStates: States + FreelyMutableState`——**可自由改**。

- **派生状态（不可手动改）**（`examples/state/computed_states.rs`）：
```rust
struct InGame;   // 零大小类型：存在即激活
impl ComputedStates for InGame {
    type SourceStates = AppState;
    const ALLOW_SAME_STATE_TRANSITIONS: bool = false;   // 防止仅 paused/turbo 变化就重跑 OnEnter
    fn compute(sources: AppState) -> Option<Self> {
        match sources { AppState::InGame { .. } => Some(Self), _ => None }
    }
}
app.add_computed_state::<InGame>();
```
`ComputedStates` **不是** `FreelyMutableState`（`crates/bevy_state/src/state/computed_states.rs:68`）——**不能 `set`**，值完全由 `compute(source)` 推出，源一变就重算覆盖。
- **ZST**（`InGame`/`TurboMode`）：无内部状态，`Some(Self)` 只表示「存在即激活」，`in_state(InGame)` 不管其它字段。
- **enum**（`IsPaused`/`Tutorial`）：有互斥分支。
- **多源派生**：`type SourceStates = (TutorialState, InGame, Option<IsPaused>)`，元组可包 `Option`（即使状态不存在也会调用 `compute`，传 `None`）。

**`ALLOW_SAME_STATE_TRANSITIONS = false` 防什么**：只改了 `turbo` 时，`AppState` 变了但 `InGame::compute` 仍返回 `Some(InGame)`（值没变）；若不设此常量，`OnEnter(InGame)` 会重跑一次（重建整个游戏场景）。`StateTransitionEvent` 仍会发出，只是 enter/exit schedule 不跑（`crates/bevy_state/src/state/transitions.rs:250`）。

### 【原理】转换时机：四段有序执行
所有状态切换集中在 `StateTransition` schedule（`transitions.rs:80`，在 `PreUpdate` 之后、`RunFixedMainLoop` 之前运行一次）：
```
StateTransition schedule
 1. DependentTransitions   // 把 NextState 落到 State<S>，并重算所有 Sub/Computed 状态
 2. ExitSchedules          // 离开状态的 OnExit，叶 → 根
 3. TransitionSchedules    // OnTransition（仅 exited/entered 都非空）
 4. EnterSchedules         // 进入状态的 OnEnter，根 → 叶
```

### 【最佳实践】
- `OnEnter/OnExit` 做一次性 setup/teardown；`run_if(in_state(...))` 做持续逻辑开关。
- 状态专属 UI/实体优先用 `DespawnOnExit` / `DespawnOnEnter`（`examples/ecs/state_scoped.rs`），别手写 cleanup。
- 层级状态用 `SubStates`；多状态组合投影用 `ComputedStates`。
- 自定义「同状态重进」用 `examples/state/custom_transitions.rs`（`OnReenter`）。
- 陷阱：依赖 `ComputedStates`/`SubStates` 的系统在状态不存在时 `Res<State<S>>` 会失败——用 `run_if(state_exists::<S>)` 或 `Option<Res<State<S>>>`。
- `NextState::set()` **不会立即生效**，下一帧 `StateTransition` 才应用。

### 【速记】
| | `States` | `SubStates` | `ComputedStates` |
|---|---|---|---|
| 可 `set`？ | ✅ | ✅ | ❌ 只能派生 |
| 何时存在 | 永远 | 源满足时，否则移除 | `compute` 返回 `Some` 时 |
| 值从哪来 | 手动设 | 手动设（首次 `should_exist` 初始化） | 每次源变化重新 `compute` |
| 注册 | `init_state` | `add_sub_state` | `add_computed_state` |
| 典型 | 顶级状态机 | 子阶段（战斗/暂停） | 正交派生（InGame/Turbo） |

- `States` = 我手动摆的档位。
- `SubStates` = 子档位，父不在时整个消失，但在的时候能手动拨。
- `ComputedStates` = 仪表读数，父一变自动重算，我碰不了；它「在不在」本身也是信息。

---

## 2.6 Reflection（反射）

### 【问题】
Rust 静态类型，如何按字符串名字访问字段、运行时构造未知类型、不写 serde 就序列化？`Reflect`/`PartialReflect`/`Dynamic*`/`TypeRegistry`/type data 关系？

### 【方案】
```rust
// examples/reflection/reflection.rs
#[derive(Reflect)]
#[reflect(from_reflect = false)]
pub struct Foo { a: usize, nested: Bar, #[reflect(ignore)] _ignored: NonReflectedValue }
// 运行期按名访问
*value.get_field_mut("a").unwrap() = 2usize;
value.apply(&patch); // 用 DynamicStruct 打补丁
```
- 各形状分支：`examples/reflection/reflection_types.rs`；动态类型：`examples/reflection/dynamic_types.rs`。
- 注册表：`TypeRegistry` / `AppTypeRegistry`；泛型需**逐个实例**注册 `.register_type::<MyType<u32>>()`（`examples/reflection/generic_reflection.rs`）。
- 序列化：`#[derive(Reflect)]` 自带 `SerializationData`，用 `ReflectSerializer`/`ReflectDeserializer`（`examples/reflection/serialization.rs`）。
- Type data：`#[reflect_trait]` / `registry.register_type_data::<T, ReflectDamageable>()`（`examples/reflection/type_data.rs`）。

### 【原理】
`Reflect: PartialReflect + Any` 是完整类型对象；`PartialReflect` 提供字段级操作。动态类型（`DynamicStruct` 等）实现 `PartialReflect` 但**不实现 `Reflect`**，用于承载编译期未知数据。`TypeRegistry` 是 `TypeId → TypeRegistration` 映射，注册时带各种 **type data**（`ReflectComponent`、`ReflectDefault`、`ReflectSerialize`…）。

### 【最佳实践】
- 要反射的字段其类型也必须 `Reflect`；不反射的用 `#[reflect(ignore)]`。
- 序列化到文件的组件/资源记得 `#[reflect(Component)]`/`#[reflect(Resource)]`。
- 泛型类型必须显式注册每个用到的实例。
- 字符串访问字段是运行时错误源，优先用 `try_*`。

---

## 相关章节
- 反射驱动的场景序列化：[场景与序列化](03-scene-and-serialization.md)
- 检查器/远程编辑依赖反射：[工具链与调试](11-tooling-and-debugging.md)
- 每实体状态 + 派生数值缓存：[14](14-per-entity-state-and-derived-stats.md)
