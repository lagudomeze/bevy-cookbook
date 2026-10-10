# 14. 每实体状态与派生数值

> [← 回到目录](README.md)
>
> 本章回答两个常见困惑：**「`States` 是全局的，那单个实体自己的状态怎么做？」** 和 **「一个数值要叠加很多来源的修正（buff/装备/基础属性），最终还要缓存，怎么设计？」**。
> 注意：这两个都不是 `States` 能解决的事——`States` 是应用级全局流程控制。

---

## 14.1 澄清：`States` 是全局的，不是每实体的

### 【问题】
`State<AppState>` 是**一个全局 Resource（单例）**，管的是「整个 App 现在处于菜单还是游戏里」。它天生不是给单个实体用的。想做「这个敌人处于 Idle/Chasing/Dead」「这个技能处于冷却中/就绪」——`States` 不对口。

### 【原理】
`States` 的机制（`OnEnter/OnExit` schedule、`StateTransition`、`NextState`）全部围绕**一个全局资源**的切换设计。Bevy 没有内置「每实体的 OnEnter/OnExit」。要表达每实体状态，用**普通组件**，并用**观察者**模拟进出回调。

---

## 14.2 每实体状态机

### 方案 A：枚举组件 + 匹配系统（最直白）

```rust
#[derive(Component, PartialEq, Clone, Copy)]
enum EnemyState { Idle, Chasing(Entity), Dead }

fn update_enemy(
    mut q: Query<(&Transform, &mut EnemyState)>,
    players: Query<&Transform, With<Player>>,
) {
    for (tf, mut state) in &mut q {
        match *state {
            EnemyState::Idle => { /* 发现玩家 → *state = EnemyState::Chasing(p); */ }
            EnemyState::Chasing(_) => { /* 追击 */ }
            EnemyState::Dead => {}
        }
    }
}
```
简单、直观、适合「状态少、逻辑集中在一个系统」。

### 方案 B：标记组件 + 观察者（模拟每实体的 OnEnter/OnExit）

每个状态是一个**标记组件**；换状态 = 移除旧标记、插入新标记；观察者在挂上/摘下时跑一次逻辑。这是最贴近 `States` 语义的每实体版：

```rust
#[derive(Component)] struct Idle;
#[derive(Component)] struct Chasing;

// 换状态
commands.entity(e).remove::<Idle>().insert(Chasing);

// 进入/离开回调（参考 examples/ecs/observers.rs 的 on_add_mine / on_remove_mine）
app.add_observer(|ev: On<Add<Chasing>>, mut q: Query<&mut Sprite>| {
    if let Ok(mut s) = q.get_mut(ev.entity) { s.color = Color::srgb(1.0, 0.0, 0.0); }
});
app.add_observer(|ev: On<Remove<Chasing>>, mut q: Query<&mut Sprite>| {
    if let Ok(mut s) = q.get_mut(ev.entity) { s.color = Color::WHITE; }
});
```

**「离开时」的另一种写法**是 `RemovedComponents<T>`（`examples/ecs/removal_detection.rs`）：
```rust
fn react_on_removal(mut removed: RemovedComponents<Chasing>, mut q: Query<&mut Sprite>) {
    for e in removed.read() { if let Ok(mut s) = q.get_mut(e) { s.color = Color::WHITE; } }
}
```

### 【原理对比】
- 观察者立即同步执行，适合「状态一变就做一次」的副作用（换色、播动画、加/减组件）。
- `RemovedComponents` 在系统里每帧检查，适合「批量处理本帧的所有移除」，但要留意它们只在移除后的有限帧内可见。
- 组件生命周期钩子（`examples/ecs/component_hooks.rs`）开销更低，但一个组件每种钩子只能有一个，适合维护索引，不适合业务回调。

### 【最佳实践】
- 状态少 → 枚举组件；状态间副作用多、需「进入/离开」语义 → 标记组件 + 观察者。
- `DespawnOnExit(State)` 只对**全局 `States`** 生效，别拿来管每实体状态；每实体用 `On<Remove<StateMarker>>` 或 `RemovedComponents` 自行清理。
- 换状态时**避免同帧既 remove 又 insert 同一逻辑**导致观察者顺序混乱；必要时拆成两步或依赖 `.chain()`。

---

## 14.3 派生数值：基础值 + 装备 + buff/debuff + 缓存

### 【问题】
你有一个数值（如攻击力），它 = 基础属性带来的提升 + 装备加成 + buff/debuff 修正，而这些输入经常变；最终希望有**一个缓存值**供其它系统读取。目录里没有现成示例——这是需要你自己设计的「stat aggregation / 派生属性」模式。

### 【方案】三层结构 + 脏标记缓存

```rust
// 1) 基础值（来源之源头）
#[derive(Component, Default, PartialEq)]
struct BaseStats { strength: f32, agility: f32 }

// 2) 修正项：加算 / 乘算 分开，来源（装备/buff）用 tag 区分
#[derive(Clone)]
struct Modifier {
    stat: StatKind,
    op: Op,                 // Flat 或 Percent
    amount: f32,
    source: ModSource,      // Base / Equipment / Buff
    duration: Option<Timer>,// buff 有时限；装备/基础为 None
}
#[derive(Component, Default)]
struct Modifiers { list: Vec<Modifier> }

#[derive(Clone, Copy, PartialEq)] enum StatKind { Attack, Defense }
#[derive(Clone, Copy)] enum Op { Flat, Percent }
#[derive(Clone, Copy, PartialEq)] enum ModSource { Base, Equipment, Buff }

// 3) 缓存输出（只读，供其它系统消费）
#[derive(Component, Default)]
struct DerivedStats { attack: f32, defense: f32 }

// 脏标记
#[derive(Component)] struct StatsDirty;
```

**打脏标记**（谁改了输入就标记）：
```rust
fn mark_stats_dirty(
    mut commands: Commands,
    changed: Query<Entity, Or<(Changed<BaseStats>, Changed<Modifiers>)>>,
) {
    for e in &changed { commands.entity(e).insert(StatsDirty); }
}
```

**重算**（只在有脏实体时跑；修正项**先加算、后乘算**）：
```rust
fn recompute_stats(
    mut commands: Commands,
    q: Query<(Entity, &BaseStats, &Modifiers), With<StatsDirty>>,
) {
    for (e, base, mods) in &q {
        let mut attack_flat = base.strength * 2.0;   // 基础属性 → 提升
        let mut attack_pct  = 0.0;
        let mut defense_flat = base.agility * 1.0;
        let mut defense_pct  = 0.0;
        for m in &mods.list {
            match (m.stat, m.op) {
                (StatKind::Attack,  Op::Flat)    => attack_flat  += m.amount,
                (StatKind::Attack,  Op::Percent) => attack_pct   += m.amount,
                (StatKind::Defense, Op::Flat)    => defense_flat += m.amount,
                (StatKind::Defense, Op::Percent) => defense_pct  += m.amount,
            }
        }
        let derived = DerivedStats {
            attack:  attack_flat  * (1.0 + attack_pct),
            defense: defense_flat * (1.0 + defense_pct),
        };
        // set_if_neq 可避免「值没变也标记 changed 引发下游抖动」
        commands.entity(e).insert(derived).remove::<StatsDirty>();
    }
}

app.add_systems(Update, (
    mark_stats_dirty,
    recompute_stats.run_if(any_with_component::<StatsDirty>),
).chain());
```

**buff 的时长处理**（每 buff 一个 `Timer`，到点移除 → 触发重算）：
```rust
fn tick_buff_durations(time: Res<Time>, mut q: Query<&mut Modifiers>) {
    for mut mods in &mut q {
        mods.bypass_change_detection().list.retain_mut(|m| match &mut m.duration {
            Some(t) => { t.tick(time.delta()); !t.is_finished() }  // 到点 → 剔除 → 重算
            None => true,
        });
        if changed { /* 用 set_if_neq 或显式标记以触发 Changed<Modifiers> */ }
    }
}
```
> 提示：`&mut Modifiers` 只要被解引用就会标记 `Changed`。若想让「本帧什么都没剔除」时**不**触发重算，用 `bypass_change_detection()` 先看有没有真的变，或给 `Modifiers` 实现 `PartialEq` + `set_if_neq`（见 `examples/ecs/change_detection.rs`）。

### 【原理】
- 变更检测 `Changed<T>` 是「相对于本系统上次运行」的 tick 比较：改动过的输入会被 `mark_stats_dirty` 捕获。
- 脏标记把「重算」与「每帧」解耦：`run_if(any_with_component::<StatsDirty>)` 让系统在没有变化时**完全不运行**；算完 `remove::<StatsDirty>()` 复位。
- 消费者只读 `DerivedStats`，绝不自己再叠修正，避免「谁负责算」的职责混乱。

### 【最佳实践】
- **修正项顺序**：先全部加算（`base + Σflat`），再统一乘算（`* (1 + Σpercent)`）。顺序错了会出现「百分比把后加的 flat 也放大」的漂移。
- **来源隔离**：装备/基础/buff 的修正都塞进同一个 `list`，用 `source` 区分，方便一键移除某一类（`list.retain(|m| m.source != ModSource::Buff)` 用于「清空所有 buff」）。
- **装备**：一件装备卸下 = 把 `source == Equipment` 且属于该装备的修正删掉、换上新的；或者每件装备是独立实体/关系（`ChildOf`/自定义 `Relationship`），穿戴卸载时增删，再打脏标记。数据量小时用 `Vec` 足够。
- **跨实体施加 buff**：施法者系统直接 `modifiers.get_mut(target).list.push(buff)`；或用观察者 `On<Add<Buff>>` 在目标上追加。两种都会改 `Modifiers` → 自动重算。
- **别把每实体的值做成 Resource**：Resource 是全局单例，会强迫你维护 map；用组件（每个实体一份）。
- **缓存的意义**：把「N 个来源的求和」从「每个消费方各算一遍」变成「一处算、多处读」，并借 `run_if` 做到「不变不重算」。
- 需要「读取时保证是最新的」时，把 `recompute_stats` 与消费系统用 `.chain()` 排序，或把消费系统放到重算之后。

### 【变体：真正的事件驱动】
若来源改动都走事件/观察者（buff 增删、装备变更），可以完全不用每帧扫 `Changed`：在对应观察者里 `commands.entity(target).insert(StatsDirty)`。示例可参考 `examples/ecs/observers.rs` 的连锁触发模式。变化少时更省。

---

## 相关章节
- 变更检测与 `set_if_neq`：[ECS 核心 · 查询](02-ecs-core.md#22-queries过滤变更检测并行不相交)
- 观察者与组件生命周期：[ECS 核心 · 资源/消息/观察者](02-ecs-core.md#23-resourceslocalmessage缓冲事件observer观察者)
- 全局状态与状态转换（对照）：[ECS 核心 · States](02-ecs-core.md#25-states--substates--computedstates)
- UI 与状态联动：[UI 与文本](07-ui-and-text.md)
