# 对象与集合（Ch4）：shape、tuple、只读

> "我的数据长啥样"的完整词汇：shape（对象/别名）、multiplicity（数组/元组）、dynamism（index signature）、permanence（readonly/as const）。

## 1. 对象形状

Inline（`const user: { name: string; age: number }`）——缺字段、多字段、类型错全报错；或 `type` 命名复用。

为什么：JS 里对象形状是 free-for-all，拼错 key 运行时才炸。typed shape=强制合同+工具红利：rename 深层字段，编译器当探照灯扫所有 consumer；autocomplete 知道每个合法 key。

三修饰符：
- **`?`** 可选（`description?: string`——在就 enforce，不在也合法；访问配 `?.`）；
- **`readonly`** 初始化后不许 reassign（id、timestamp、DB key）——**编译期保护，运行时该变还能变**；
- **嵌套**逐层建模，optionality 每层可组合（notifications 可选，但一旦在，email/sms 必填）。

坑：`readonly` 拦 reassign 不拦引用对象的 mutation，且运行时蒸发。shape 在两个以上签名出现→抽 named type。

## 2. Type alias + index signature

`type` 能命名**任何**类型表达式：字面量 union（`type UserRole = "admin" | "editor" | "viewer"`——`"Admin"` 直接拒，autocomplete 就位）、函数签名（`type Logger = (msg: string, level: string) => void`——回调/注入逻辑的可复用合同，DI 风格的 report 函数端到端类型安全）。

**Index signature**：知道值规则不知道 key——`type ScoreMap = { [playerId: string]: number }`（"string key、number 值"的 map；用户分数、动态配置、运行时生成的表）。心智模型："这是个 map——key 不 care，值必须守规矩"。

**硬规则**：一加 index signature，所有属性（含显式命名的）必须服从值类型。`debugMode: boolean` 配 `[key: string]: string` 直接编译错误。要混值→别用 index signature，逐个字段建模。`Record<string, T>` 和 `{ [key: string]: T }` 等价，看团队风格。

## 3. Array vs tuple

**Array**（`string[]`）：同质列表的合同——`tags.push(42)` 编译期拦，不再是静默腐坏。函数签名里锁输入输出合同。

**Tuple**（`[string, number]`）：位置有意义——"恰两个，name 然后 score，按序"。位置类型错报错，多一个元素报错（`no element at index '2'`）。"多返回值不想造对象"的轻量答案：`parseUser(input: string): [string, number]`。

决策：同类 uniform 列表（错误信息/id/邮箱）→ array；每 slot 意义不同的定结构（坐标、name/score 对、`[string, boolean]` 状态结果）→ tuple。

**Labeled tuples**（`[name: string, score: number]`）：给位置加语义名，零运行时成本，签名和编辑器提示里出现。`[string, number]` 让人猜就换它。

## 4. readonly vs as const：两样工具

- **`readonly`**：类型级修饰——`readonly string[]` 禁 push/pop，readonly tuple 禁 index reassign。是"**经这个类型看过去**不许碰"。
- **`as const`**：值级断言——`['red','green'] as const` 推最窄类型：字面量 `["red","green"]` 的 readonly tuple，**阻止 widen**（`const colors = [...]` 否则 widen 成 `string[]`）。

为什么：可变结构是静默 bug 工厂（函数改了调用者的数组；config 常量被后面 refactor 覆盖）。对象上 `as const` 双杀：每属性字面量类型 + 全 readonly——config/常量/discriminated unions 的 discriminant 值神器。

**Assignability 背下来**：mutable 可 assign 给 readonly target（经那个引用失去写权限）；readonly **永不** assign 给 mutable——TS 拦降级（会重开 mutation 门）。两者都只编译期，运行时不 freeze。

Takeaway：要"值和形状都锁死"用 `as const`（更强）；type annotation 里想"接受数组但承诺不改"用 `readonly string[]`（签名即文档）。

## 自测 4 问（带答案要点）
1. `?` / `readonly` / 嵌套各管什么？readonly 的坑？
   → 可选/禁 reassign/逐层建模；只编译期，不拦 mutation。
2. Index signature 硬规则？`Record<string,T>` 关系？
   → 所有属性服从值类型；等价，风格问题。
3. Array vs tuple 决策规则？labeled tuple 干嘛？
   → 同质列表 vs 位置有意义；给位置加语义名零成本。
4. readonly vs as const？assignability 方向？
   → 类型级"看过去不许碰" vs 值级"锁最窄"；mutable→readonly 可，readonly→mutable 永不。
