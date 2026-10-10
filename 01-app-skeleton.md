# 01. 应用骨架：App / Plugin / Schedule / System / Time

> [← 回到目录](README.md)

## 1.1 App、Plugin、启动流程

### 【问题】
`App` 是什么？`Plugin` 和直接 `app.add_systems(...)` 有何区别？`Startup` 为何只跑一次？无窗口环境怎么跑？

### 【方案】
- 最小 app：`App::new().run()`（`examples/app/empty.rs`）；带渲染/窗口用 `DefaultPlugins`（`examples/app/empty_defaults.rs`）。
- `Plugin` 本质是「把一批 `App` builder 调用打包」：

```rust
// examples/app/plugin.rs
impl Plugin for PrintMessagePlugin {
    fn build(&self, app: &mut App) {
        app.insert_resource(state)
           .add_systems(Update, print_message_system);
    }
}
```
- 多个插件组成 `PluginGroup`，可 `PluginGroupBuilder::start::<Self>().add(...)`、`.disable::<T>()`、`.add_before::<T>(...)`（`examples/app/plugin_group.rs`）。
- 无窗口：`DefaultPlugins.build().disable::<WinitPlugin>()`（`examples/app/without_winit.rs`）；纯 headless 用 `ScheduleRunnerPlugin::run_once()` / `run_loop(Duration)`（`examples/app/headless.rs`）。
- 自定义循环：`.set_runner(my_runner)`，runner 里先 `app.finish(); app.cleanup();` 再反复 `app.update()`（`examples/app/custom_loop.rs`）。

### 【原理】
`App` 持有 `World` + `Schedules`。`Plugin::build` 只是推迟到构建期执行的一批注册。`Main` schedule 按 `MainScheduleOrder` 依次运行 `First → PreUpdate → StateTransition → RunFixedMainLoop → Update → … → Last`（`crates/bevy_app/src/main_schedule.rs`）。窗口 app 的 `App::run()` 会进入 winit 事件循环——**在 iOS/Web 上永不返回**，所以不要在其后写代码（`examples/app/return_after_run.rs` 专门说明）。

### 【最佳实践】
- 功能尽量封装成 plugin，作用域越小越好，便于复用与开关。
- `Startup`（= `PreStartup→Startup→PostStartup`）只做初始化；每帧逻辑放 `Update`；确定性逻辑放 `FixedUpdate`。
- `AppExit` 实现了 `Termination`，可直接作 `main` 返回类型（`examples/ecs/ecs_guide.rs`）。
- 陷阱：全局只能注册一次的插件（如 `LogPlugin`）重复添加会 panic，用 `.disable::<LogPlugin>()` 规避（`examples/app/headless.rs`）。

---

## 1.2 系统、调度、SystemSet、排序、运行条件

### 【问题】
系统默认并行、顺序不定。怎么声明依赖？`chain`/`before/after`/`SystemSet` 怎么配合？`run_if` 怎么组合？

### 【方案】（`examples/ecs/ecs_guide.rs` 是最佳总览）
```rust
#[derive(SystemSet, Debug, Hash, PartialEq, Eq, Clone)]
enum MySystems { BeforeRound, Round, AfterRound }

.configure_sets(Update, (MySystems::BeforeRound, MySystems::Round, MySystems::AfterRound).chain())
.add_systems(Update, (
    (new_round_system, new_player_system).chain().in_set(MySystems::BeforeRound),
    score_system.in_set(MySystems::Round),
    (score_check_system, game_over_system.after(score_check_system)).in_set(MySystems::AfterRound),
))
```
运行条件（`examples/ecs/run_conditions.rs`）：`.run_if(resource_exists::<T>)`、`.run_if(in_state(S))`，可 `.and_then()` / `.or_else()`，配 `not(...)`。自定义条件就是「返回 bool 的只读系统」。

自定义 schedule（`examples/ecs/custom_schedule.rs`）：`Schedule::new(Label)` + `app.add_schedule(...)`，再用 `MainScheduleOrder::insert_after(Update, MyLabel)` 插入主循环。自定义执行器见 `examples/ecs/custom_executor.rs`。

### 【原理】
每个 `Schedule` 构建时生成依赖图：系统间的**数据访问冲突**（`Res`/`ResMut`/`Query<&mut T>`）自动产生「不可并行」约束；兼容访问则允许贪心并行。`chain()` 等价于逐个 `.before()`。`set_build_settings(ScheduleBuildSettings { ambiguity_detection: LogLevel::Warn, .. })` 可检测「顺序歧义」。

### 【最佳实践】
- 只要两系统访问同一可变数据且顺序影响结果，就显式 `.before/.after/.chain`，别依赖「碰巧」。
- `SystemSet` 做粗粒度分层，`chain` 做细粒度排序。
- `examples/ecs/nondeterministic_system_order.rs` 强调：**内部歧义也会被报告**；只有确属误报才用 `.ambiguous_with(sys)` 并写注释。
- 修改 `MainScheduleOrder` 必须在 `main()` 里做，不能在 schedule 内的系统里改。

---

## 1.3 Time / FixedUpdate / Timers

### 【问题】
`Time` 有 Real/Virtual/Fixed 三个「时钟」分别是什么？`FixedUpdate` 为何可能一帧跑 0 次或多次？定时器怎么用？

### 【方案】（`examples/time/time.rs` 最清晰）
```rust
app.insert_resource(Time::<Virtual>::from_max_delta(Duration::from_secs(5)))
   .insert_resource(Time::<Fixed>::from_duration(Duration::from_secs(1)));
```
`PreUpdate` 读 `Time<Real>`，`FixedUpdate` 读 `Time<Fixed>`，`Update` 读 `Time<Virtual>`（默认别名 `Time`）。定时器：
```rust
// examples/time/timers.rs
timer.tick(time.delta()).just_finished();
countdown.main_timer.fraction(); countdown.percent_trigger.pause();
```
`Timer::from_seconds(t, TimerMode::Once | Repeating)`，方法 `is_finished()/just_finished()/fraction()/pause()`。

### 【原理】
三层时钟：`Real`=OS 真实时间；`Virtual`在 Real 上做缩放/暂停/`max_delta` 钳制（默认 `Time` 即 Virtual）；`Fixed`以固定步长累积。`RunFixedMainLoop` 每帧按虚拟时间累积并执行 `FixedMain` 循环，因此**一帧内 `FixedUpdate` 可能 0..N 次**（catch-up），`overstep()` 是未消耗的累积时间。

### 【最佳实践】
- 表现/动画/输入用 `Update`+`Time<Virtual>`（可暂停/变速）；物理/确定性规则用 `FixedUpdate`+`Time<Fixed>`。
- 每个 `Timer` 必须**恰好 tick 一次/帧**；用 `just_finished()` 捕捉「刚结束」这一刻。
- 用 `Time<Virtual>::from_max_delta` 限制单帧最大步长，避免卡顿后时间跳变。

---

## 相关章节
- 状态与状态转换发生在哪个时机：[ECS 核心 · 状态](02-ecs-core.md#25-states--substates--computedstates)
- 用 timer 做 buff 时长：[每实体状态与派生数值](14-per-entity-state-and-derived-stats.md)
