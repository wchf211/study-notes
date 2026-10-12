# 类型系统（Ch3）：推断、注解、any/unknown、读懂报错

> 本章是内功：让"推断/注解/收窄"变成本能，后面 unions/generics 全是词汇不是新概念。

## 1. Inference：不写类型也有类型

`let score = 100` → number；`let user = "Ada"` → string。**推断是 sticky 的**：推成 number 后再赋 `"six"` 就是编译错误——类型是变量此后遵守的规则，不是当前值的描述。

**let vs const 的关键区别**：`let mode = "dark"` 推成宽泛 string（可能变）；`const theme = "dark"` 推成字面量 `"dark"`（永不变）。这区别后面在 literal types/discriminated unions 里是承重的——**能 const 就 const**。

坑：`let data;`（没 initializer）或 `let result = null` → 推断塌成 **any**，检查全关。别拿 null/undefined 当"我一会填"的占位——它们不带类型信息。

## 2. Annotations：推断瞎的时候你说话

`let username: string`、`function greet(name: string): string`。不改变运行时，只改变编译器 enforcement。

三处必须/应该写：未初始化变量（`let result: string` 先锁类型）、**函数边界**（strict 下无 annotation 参数=implicit any 错误）、建模缺席（`let error: string | null = null`——"故意空，一会有 string"，不丢安全；`userId?: string` 可选参数，optional 是设计意图）。

**Structural typing 前瞻**：TS 看形状不看名。`let greet: (name: string) => string` 接受任何签名兼容的函数。兼容不对称但有用：参数少的函数可 assign 给参数多的（多余参数被忽略——回调忽略 index 参数同理）；返回 string 的可满足 `() => void`（返回值被丢弃）；反过来 void→string 不行。

Return type 指导：TS 多数能推，但** exported/shared 函数故意标**——文档意图、double-check 逻辑、抓漏 return。纯副作用函数用 `void`。经验：**参数必标；shared 函数的返回认真考虑标**。

## 3. any vs unknown：放弃安全 vs 推迟安全

**any**：全开逃生舱——assign 啥都行、当函数调、随便访问属性，编译器啥都不查，错全变运行时炸。
**unknown**：安全版"我不知道"——啥都能 assign 进来，但**证明它是什么之前不许用**（narrowing 或 assertion）。

这是给不确定数据（API 响应、localStorage、表单输入）的正确类型——**"先证明再碰"的编译期纪律**。

**Narrowing 工具箱**（TS 懂的运行时检查）：
- `typeof` 缩 primitives（+区分 function/object）；
- `instanceof` 缩 class 实例（Date  canonical）；
- `in` 按属性存在缩（`"length" in obj`）。
- **背下来**：`typeof null === "object"` 是 JS 历史 quirk，object 检查必配 `!== null`。
- **Custom type guards**：返回 type predicate 的函数（`pet is Dog`），教编译器结构检查（`"bark" in pet`），之后分支内自动 narrow。

**Assertions vs conversions（关键区别）**：
- `value as string`（JSX 外 `<string>value` 也行，prefer `as`）= 你告诉编译器"信我"——**运行时啥都不变**，错了就编译通过+运行时炸。合法用：DOM（`querySelector` 返回 `Element | null`，assert 成 `HTMLButtonElement` 解锁 `.disabled`）、无类型第三方库、tricky 推断。
- **`!` non-null assertion**="信我，非 null"——只在确定时用（如静态 DOM）。
- 类型对但形状不对（手里是 `"42"` 要 number）→ 真**运行时转换**：`Number()`/`String()`/`Boolean()`。背 mantra：**assertions 不改变值，conversions 才变**。附赠：`Boolean("false")` 是 `true`（非空串 truthy）。

## 4. 五大报错（读懂=debug 速度）

1. **"Type 'X' is not assignable to type 'Y'"**——值和期望对不上。看意图改值或改 annotation，别 silence。
2. **"Object is possibly 'undefined'"**——碰了可能缺席的值（strictNullChecks）。修：显式 if guard / optional chaining（`str?.length`）/ fallback。这是编译器在拦 `Cannot read property of undefined`。
3. **"Property 'X' does not exist on type 'Y'"**——typo/假设错/类型不全。maybe-present 属性用 `in` guard narrow。
4. **"Variable implicitly has an 'any' type"**——noImplicitAny 拒绝猜。修：annotation，或 unknown+guard。
5. **"Argument of type 'X' is not assignable to parameter of type 'Y'"**——call site 传错。转换输入或修源头。

Workflow：读编译器期望什么 → 判断**值错还是类型错** → 只改一边。**永远别拿 any 灭红**。

## 自测 4 问（带答案要点）
1. let vs const 在推断上差在哪？为什么重要？
   → let  widen 到 string，const narrow 到字面量；后面 literal/discriminated unions 承重。
2. any vs unknown？uncertain 数据默认哪个？
   → 放弃安全 vs 推迟安全；默认 unknown + narrowing。
3. Assertion vs conversion？`!` 什么时候用？
   → 前者只骗编译器不改值，后者真转；`!` 只在确定非 null 用。
4. 看到 "possibly undefined" 三种修法？
   → if guard / `?.` / fallback；本质是 strictNullChecks 在拦运行时炸。
