# 类型组合（Ch5）：union、discriminated、interface、satisfies

> 本章是**类型设计**不是语法：variation（either/or）、combination（both/and）、evolution（extends）、precision（literals）——让编译器 enforce 正确和完备。

## 1. Literal + union：值即类型

`"dark"`、`42`、`true` 都能当类型——"恰是这个值"。`let mode: "dark"` 拒 `"light"`。Union（`|`）："是这几个之一"，如 `"admin" | "editor" | "viewer"`。

为什么：字面量把运行时校验逻辑变成编译期法律——config mode、feature flag、状态值拼不错。配 `as const`，整个对象冻成字面量类型（`typeof config` 让 mode 保持 `"dark"` 不 widen 成 string）。

Union 的代价：编译器只知道"可能是几个"，**narrow 前啥都不假设**——`string | number` 上调 `.toUpperCase()` 必须先证是哪边。`typeof` 缩 primitives；对象 union 用 custom type guard（`pet is Dog` + `"bark" in pet` 的真运行时检查）。**Guard 的诚实=它的运行时检查；撒谎的 guard 比没 guard 坏**。

**Function overloads**：参数合法组合依赖另一个参数的值时用——多个 call signature（`handleRequest("GET"|"DELETE"): string` / `handleRequest("POST"|"PUT", body: string): string`）+ 一个兜底的 implementation signature（内部用，调用者看不见）。直觉：union 参数不够表达"哪个组合合法"时上 overload。

## 2. Discriminated unions + exhaustive：旗舰 pattern

**Discriminated union**：一组对象类型共享一个字面量字段——**discriminant**（`type`/`kind`/`status`）。`result.type === "success"` 后整个对象自动 narrow 到 Success 形状——无 guard 无 cast，编译器全包。`switch` discriminant 多 variant 干干净净。

为什么：API 响应、状态机、渲染 variant——**加新 case 不会静默漏**。exhaustive 技巧：default 分支里 `const _exhaustive: never = result`。全处理了→剩 `never`，编译过；加了 Pending 忘写 case→default 里还是 Pending，assign 给 never 炸——**编译器 enforce 完备性**。直觉：`never`="这不可能发生"，assign 它=断言"我全处理了"。

设计指导：**一个字段决定其余形状**时就上——API 响应、async 三态（loading/success/error）、UI variant。+never 检查=编译期合同。

## 3. Interface vs type：背决策表

**interface**：命名对象合同（字段/?/readonly），可用在变量/参数/返回/`implements`。杀手锏：
- **extends**：`interface UserResponse extends BaseResponse` 加字段；可 extend 多个 base，甚至对象形的 type alias。**硬限制**：不能 extend union/primitive（只认对象形；对象 intersection 行）。
- **Declaration merging**：同名 interface 声明两次自动合并——大模块化代码库、augment 第三方类型的神器。

决策表（背下来）：
| | interface | type |
|---|---|---|
| extends/声明合并 | ✅ | ❌ |
| `\|` union / `&` 直接写 / 命名 primitive | ❌ | ✅ |
| class implements | ✅（对象形） | ✅（对象形） |

经验：**对象形状且会演化/被 extend/跨团队共享→interface**；union/组合/非对象→type。每个 interface 都能当 type 用，反过来不行。

## 4. Intersection：both/and 组合

`&` 是"全都要"（union 是"二选一"）。能力 mixin：小而专的接口（Drivable/Floatable/Locatable）拼成 AmphibiousVehicle，不用继承层级。比 extends 野：啥都能拼（interface、alias、inline、utility type）。

**冲突语义背下来**：同名同类型合并没事；一边是另一边的子类型→窄的赢（`number & 42 → 42`）；不兼容→塌成 `never`（`{id: string} & {id: number}` 的 id 是 never——啥都 assign 不进，全报错）。`string & number` 也是 never——"这类型不存在"。

代数是集合式的，重构可依赖：交换/结合律、对 union 分配（`(A|B)&C=(A&C)|(B&C)`）。`"hello" & string → "hello"`（字面量保留，高级 narrowing 有用）。

用途：分层约束的组合工具——`WithId<T> = T & { id: string }` 给形状加能力不动原形状。

## 5. satisfies：形状校验 + 字面量保留

老三样各瘸一条腿：`: Type` 注解校验结构（抓 typo/缺 key）但 **widen 字面量**；`as Type` 更糟——跳过 excess-property 检查还 widen；`as const` 字面量完美但**不校验**（typo 滑过去）。**satisfies 全都要**：`const config = {...} satisfies AppConfig`——每个 key/value 按合同查，推断的字面量类型不动。

对照（缺 key / 多 key / 保字面量）：`: Type`→yes/yes（仅对象字面量）/no；`as Type`→no/no/no；`as const`→no/no/yes；**satisfies**→yes/yes/yes。

用途：config 对象、路由表、feature-flag 字典——**结构要对，字面量要准**（下游 narrowing+autocomplete 靠它）。idiom：先定合同 type，写字面量，尾巴 `satisfies`。纯编译期，不改变推断类型。

## 自测 4 问（带答案要点）
1. Discriminated union 三件套？never 检查的原理？
   → 共享 discriminant 字面量字段/switch narrow/default 里 assign never；never="不可能"，assign 炸=有 case 漏了。
2. Interface vs type 决策表？
   → 演化/extend/合并→interface；union/组合/primitive→type。
3. `&` 冲突三规则？
   → 同类型合并；子类型窄赢；不兼容塌 never。
4. satisfies 解决什么 dilemma？四方案对照？
   → 校验结构+保留字面量；`: Type`/as/as const/satisfies 的 yes-no 表。
