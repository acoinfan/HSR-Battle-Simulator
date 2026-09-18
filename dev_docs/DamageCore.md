1. 核心数据结构

先定义一次伤害行为：

struct DamageAction {
    DamageMode mode;

    // 这次行为涉及的攻击段
    std::vector<DamageHit> hits;
};

每一段：

struct DamageHit {
    // 当前这一段使用的公式参数
    FormulaInput formula_input;

    // 是否允许暴击
    bool can_critical = false;
    double crit_rate = 0.0;
    double crit_damage = 1.5;
};

于是一个简单三段攻击就是：

DamageAction
├── DamageHit 1
├── DamageHit 2
└── DamageHit 3
2. DamageCore

接口可以先定成：

class DamageCore {
public:
    DamageActionResult resolve(
        const DamageAction& action,
        RandomContext& rng
    ) const;

private:
    DamageResult resolve_hit(
        const DamageHit& hit,
        DamageMode mode,
        RandomContext& rng
    ) const;
};

核心逻辑：

DamageActionResult DamageCore::resolve(
    const DamageAction& action,
    RandomContext& rng
) const {
    DamageActionResult result;

    for (const auto& hit : action.hits) {
        DamageResult damage =
            resolve_hit(
                hit,
                action.mode,
                rng
            );

        result.hits.push_back(damage);
        result.total_damage += damage.damage;
    }

    return result;
}

而单段：

DamageResult DamageCore::resolve_hit(
    const DamageHit& hit,
    DamageMode mode,
    RandomContext& rng
) const {
    bool critical = false;

    if (hit.can_critical) {
        critical =
            rng.roll_probability(hit.crit_rate);
    }

    double damage =
        DamageFormula::calculate(
            mode,
            hit.formula_input,
            critical
        );

    return DamageResult{
        .damage = damage,
        .critical = critical
    };
}
3. 但是这里有一个很重要的调整

我们之前讨论过：

第 10 段可能依赖前 9 段。

那么 std::vector<DamageHit> 不能作为最终设计，因为它意味着所有 Hit 在行为开始之前就已经确定。

所以我更建议把上面的结构理解成第一版简单 Core，最终 Core 应该允许：

DamageCore
    │
    ├── 解析当前状态
    │
    ├── 生成 Hit
    │
    ├── Formula
    │
    ├── 应用当前结果
    │
    ├── 更新行为内部状态
    │
    └── 生成下一 Hit

也就是说：

while (has_next_hit()) {
    DamageHit hit = generate_next_hit(state);

    DamageResult result =
        resolve_hit(hit, rng);

    apply_result(result, state);

    update_execution_state(result);

    if (!should_continue()) {
        break;
    }
}
4. 因此我建议最终 Core 有三个概念
DamageAction

描述：

“我要执行什么伤害行为？”

例如：

struct DamageAction {
    DamageMode mode;
    EntityId source;
    std::vector<EntityId> targets;

    // 技能/行为自己的参数
    ...
};
DamageExecutionState

描述：

“这次伤害行为已经执行到哪里了？”

例如：

struct DamageExecutionState {
    int hit_count = 0;
    int successful_hits = 0;

    double total_damage = 0.0;

    bool finished = false;
};

以后可以增加：

int buff_consumed;
int target_death_count;
double accumulated_damage;
DamageCore

负责：

DamageAction
      +
BattleState
      +
DamageExecutionState
      ↓
   DamageCore
      ↓
逐次生成 Hit
      ↓
DamageFormula
      ↓
更新执行状态
      ↓
最终 DamageActionResult
5. 最终接口可以先这样定

我比较推荐这一版作为你项目的正式方向：

class DamageCore {
public:
    DamageActionResult resolve(
        DamageAction& action,
        BattleState& state,
        RandomContext& rng
    );

private:
    DamageHit build_next_hit(
        const DamageAction& action,
        const BattleState& state,
        const DamageExecutionState& execution
    ) const;

    DamageResult resolve_hit(
        const DamageHit& hit,
        DamageMode mode,
        RandomContext& rng
    ) const;

    void apply_result(
        const DamageResult& result,
        BattleState& state
    ) const;

    bool should_continue(
        const DamageAction& action,
        const BattleState& state,
        const DamageExecutionState& execution
    ) const;
};

不过第一版实现时可以砍掉 BattleState 和复杂的 ExecutionState，先做：

DamageAction
    ↓
DamageCore
    ↓
for each Hit
    ↓
DamageFormula
    ↓
DamageActionResult

等你开始实现真实的多段技能，再把动态状态接进去。

最终职责边界
┌──────────────────────────────────┐
│           DamageCore             │
│                                  │
│  ① 解析伤害行为                  │
│  ② 决定有多少次伤害              │
│  ③ 决定每次伤害的顺序            │
│  ④ 准备 FormulaInput             │
│  ⑤ 调用 DamageFormula            │
│  ⑥ 处理每次伤害后的行为状态      │
│  ⑦ 汇总 DamageResult             │
└────────────────┬─────────────────┘
                 │
                 ▼
        ┌─────────────────┐
        │ DamageFormula   │
        │                 │
        │ 只负责：        │
        │ “这一笔是多少？”│
        └─────────────────┘

这个划分我觉得已经比较稳定了：**Core 是行为层，Formula 是数学层。**后面即使出现极其复杂的“100 段伤害 + 中途死亡 + Buff 消耗 + 额外触发”，也不会污染 DamageFormula。