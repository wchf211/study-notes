# 类与泛型（Ch6）：结构+边界、复用不丢类型

> 两根柱子：类回答"这个实体带什么保证"，泛型回答"逻辑怎么跨类型复用还不丢类型"。

## 1. Typed classes：三保证

相对 JS 类：字段必须初始化（strict 下声明了没赋值直接报错——"has no initializer and is not definitely assigned"）、构造器/方法全类型检查、**访问修饰符编译期 enforce 封装**。

初始化两式：inline（`startTime: number = Date.now()`）或构造器里。规则的存在理由：**不让你 new 出半成品对象**（运行时 undefined 潜伏）。

- `public`（默认，到处）、`private`（类内）、`protected`（类+子类）；
- **Parameter properties**：`constructor(public x: number, public y: number)`——声明+赋值一笔，构造参数直接映射字段时用。

修饰符是设计沟通：public=API 面，private=内部完整性，protected=扩展缝。
**坑**：private/protected 只编译期——运行时不存在（和 JS `#private` 真运行时私有两回事，边界要清楚）。

## 2. 继承与抽象类

`extends` 复用+特化，编译期检查：子类构造器**先 `super()` 再碰 `this`**；override 方法类型兼容（参数可更少——多态时多余参数被忽略；返回必须兼容，保证 substitutability）。

**Abstract class**：定合同但不许实例化——`abstract generate(): string`（没 body，子类必须实现）+ 具体共享方法（如 `log()`）。非抽象子类必须实现全部 abstract 成员，编译器盯着。**多态**自然来：`BaseReport[]` 里装 PDFReport/ExcelReport，调共享方法，行为按子类走，代码只依赖 base 合同。

Abstract class vs interface：**要共享具体行为+强制实现→abstract class**；只要形状合同→interface。
设计注：能组合（intersection/generics）别深继承——课里后面也往这推；但真 is-a 层级存在时，TS 让它安全。

## 3. Generics：类型参数

**泛型=类型占位符**，复用逻辑不丢类型信息。`identity<T>(value: T): T`——string 进 string 出，调用点零 annotation，T 从用法推。

**Constraints**（`T extends { id: string }`）："至少长这样"——在函数里安全操作所需的最小结构。常见：属性形状、字面量、现成 interface、union（`T extends string | number`）、array-like（`T extends any[]`）、**类型参数约束类型参数**（`U extends T`——"至少和 T 一样具体"）。多参数表关系：`merge<A,B>(a: A, b: B): A & B` 返回 intersection，全类型化。

不止函数：`interface Box<T>`（API 响应、wrapper 的形状合同）、`type Result<T>`（union/intersection/mapped 在这读着最顺）、`class Container<T>`（store/queue/repository 的 stateful 类型抽象）。**实例化后 `Box<number>` 是固定类型——泛型是编译期模板，不是运行时多态**。

Trade-off：**永远单类型的别泛型化**——泛型买灵活付复杂。泛型擅长结构，深层类型逻辑是 keyof/infer/conditional 的活（下章）。

## 4. 泛型设计：真实代码库的词汇

内置先：`Array<T>`、`Promise<T>`（resolve 出啥）、`Record<K,V>`（K key→V 值的对象）。

Pattern：
- **`ApiResponse<T> = { status: number; data: T; error?: string }`**——所有 endpoint 一个 wrapper，外层定死内层可变；
- **Decorator types via intersection**：`WithId<T> = T & { id: string }`、`WithTimestamp<T>`、`WithMeta<T,M>`——给现有形状叠 enrichment 不重写；可 compose（`WithMeta<WithId<WithTimestamp<...>>>`）；
- **Default type params**（`type Result<T = string>`）——80% 场景开箱，override 留着。

Heuristics（field-tested）：
- **别一上来就泛型**——先写几个 concrete，pattern 出来了再抽象；
- 常用处给 default；
- 紧 scope 短名（T/U/K），宽 API 描述名（Payload/Meta/Flags）；
- union/intersection/mapped 组合→type alias；要 extend/implement 的形状→interface；
- **泛型应该让签名更好读**——比两个 concrete overload 还难读就是 overengineering。

合成：`Container<T>` 类封类型化状态 + `ApiResponse<T>` 流经 class service——类管"保证"，泛型管"适应"。

## 自测 4 问（带答案要点）
1. TS 类的三保证？parameter properties 干嘛？
   → 初始化强制/构造方法类型检查/访问修饰符；构造参数直映射字段一笔写。
2. Abstract class vs interface？override 的兼容规则？
   → 共享行为+强制实现 vs 纯形状；参数可少返回须兼容。
3. `T extends {id: string}` 是什么？`Box<number>` 实例化后是什么？
   → 最小结构约束；固定类型（编译期模板）。
4. 泛型设计的四条 heuristics？什么时候别用泛型？
   → 先 concrete 再抽象/default/命名/type-vs-interface；签名变难读就别用。
