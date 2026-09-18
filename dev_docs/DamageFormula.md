DamageFormula 设计总结
1. 核心定位

DamageFormula 是伤害数学计算模块，负责根据伤害模式和传入的数据计算一次伤害的数值。

Debuff、技能、场景等在上层被解析为计算所需的数值，Formula 不直接依赖具体的 Debuff 对象或战斗系统。

2. 伤害模式

暂定六种 DamageMode：

enum class DamageMode {
    Normal,          // 普通型
    DamageOverTime,  // 持续伤害
    SelfDebuff,      // 自身 Debuff 结算
    Environment,     // 场景伤害
    SelfSkill,       // 技能对自身伤害
    Elation          // 特殊欢愉伤害
};

这些模式用于选择不同的结算逻辑，而不是要求所有模式共享完全相同的数学公式。

3. 统一的数值输入

不为每种 Debuff 单独设计输入结构，而是使用通用的 FormulaInput：

struct FormulaInput {
    double base = 0.0;
    double multiplier = 1.0;

    double source_hp = 0.0;
    double source_max_hp = 0.0;

    double target_hp = 0.0;
    double target_max_hp = 0.0;

    double stack = 0.0;
    double extra = 0.0;
};

具体公式只读取自己需要的字段。

例如：

D=HP
max
	​

×0.05

只需要使用：

input.source_max_hp
input.multiplier

不需要为“最大生命值百分比伤害”额外创建一个专属类。

4. Formula 的调用方式
class DamageFormula {
public:
    static double calculate(
        DamageMode mode,
        const FormulaInput& input,
        bool critical
    );
};

内部使用 switch：

double DamageFormula::calculate(
    DamageMode mode,
    const FormulaInput& input,
    bool critical
) {
    switch (mode) {
        case DamageMode::Normal:
            return calculate_normal(input, critical);

        case DamageMode::DamageOverTime:
            return calculate_dot(input);

        case DamageMode::SelfDebuff:
            return calculate_self_debuff(input);

        case DamageMode::Environment:
            return calculate_environment(input);

        case DamageMode::SelfSkill:
            return calculate_self_skill(input, critical);

        case DamageMode::Elation:
            return calculate_elation(input, critical);
    }

    throw std::invalid_argument("Unknown damage mode");
}

暂时不引入复杂的继承体系、IFormula 或多态公式类。

5. 与 DamageCore 的职责边界

模块

	

职责




DamageCore

	

参数校验、暴击判定、调用 Formula、组织结果




DamageFormula

	

根据 DamageMode 执行数学计算




FormulaInput

	

提供当前一次计算所需的数值




DamageInputBuilder（未来）

	

从战斗状态、Buff、Debuff 中提取并构造输入




AttackExecutor（未来）

	

管理多段伤害、触发顺序和状态变化

调用流程：

BattleState / Debuff 数据
        │
        ▼
构造 FormulaInput
        │
        ▼
DamageCore
        │
        ▼
DamageFormula::calculate()
        │
        ├── switch(DamageMode)
        │
        ▼
计算伤害数值
6. 最终设计原则

伤害模式负责选择结算逻辑。

通用数值输入负责传递公式所需的数据。

Debuff 本身是数据，不需要被 Formula 直接理解。

Formula 只计算数值，不修改 HP、不消耗 Buff、不管理触发过程。

当前使用 DamageMode + switch，避免过早复杂化。

如果未来出现大量无明确语义的 extra、value1 等字段，再针对实际问题调整输入设计。

一句话概括：

DamageCore 负责组织一次伤害结算，DamageFormula 根据 DamageMode 和通用数值输入执行具体数学公式。