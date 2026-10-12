# 实战 TypeScript（Ch7）：utility、keyof、类型编程、装饰器、迁移

> 从"写类型"到"工程类型"：建模、变换、推断——让生产代码随 domain 演化还不出错。

## 1. Utility types：类型的函数式编程

编译期函数，变换现有对象类型不重写。四大：
- **`Pick<T,K>`**：取子集——`PublicUser = Pick<FullUser,"id"|"name">`（ public 视图，passwordHash 留里面）；
- **`Omit<T,K>`**："除了那个全要"——`ClientConfig = Omit<Config,"apiKey">`；
- **`Partial<T>`**：全可选——草稿/PATCH（`UpdatePayload = Partial<Omit<User,"id">>` 只发 `{name}`）；
- **`Required<T>`**：全必填——提交前校验（`FinalForm = Required<FormDraft>`）。

为什么：domain 模型有生命周期（草稿/提交、内部/公开、完整/patch）——手写每个 variant 形状 drift。**utility 派生=单一 source of truth**；可 compose（`Record<string, Partial<Omit<Product,"sku">>>`——SKU 受保护的可编辑草稿字典）。

知道存在即可：`Readonly<T>`、`NonNullable<T>`（校验后去 null）、`Exclude<T,U>`/`Extract<T,U>`（union 过滤）、`ReturnType<T>`/`Parameters<T>`（从函数签名扒类型，跨调用边界 DRY）、`Awaited<T>`（拆 Promise）、`Record<K,V>`。

## 2. keyof + indexed access + typeof：反射三件套

从现有类型/值**派生**类型，不硬编码。
- **`keyof T`**：顶层属性名 union（`"id"|"name"`）；数组→`number|"length"|"push"|...`；tuple→`0|1|"length"|...`；`keyof unknown`=never（啥都不保证），`keyof any`=`string|number|symbol`；
- **`T[K]` indexed access**：掏属性类型（`User["name"]`→string）；`User[keyof User>`=所有值类型的 union；
- **`typeof`**（类型级）：抓值的类型（`type Settings = typeof settings`）——只接受标识符/属性链，不接受任意表达式。

**Canonical 组合**：`function getValue<T, K extends keyof T>(obj: T, key: K): T[K]`——全类型安全 getter：非法 key 编译错，返回类型恰是属性类型，零 annotation/cast。类型安全表单更新器、config accessor、ORM-ish API 全是这个 pattern。

Subtlety：keyof 只顶层（不递归）；反映 TS 静态声明不是运行时完整原型链；tuple key 含位置数字字面量，数组是通用数字索引。

## 3. Template literals + mapped types：生成类型

**Template literal types**：`` `api_${Status}` `` 把 `"pending"|"success"|"error"` 展开成 `"api_pending"|"api_success"|"api_error"`——命名规则编码一次，不用列 variants。错误码、endpoint、事件名、config key——**字符串有结构的地方**。

**Mapped types**：整个对象形状变换——`{ readonly [K in keyof User]: User[K] }` 一句话全 readonly；`[K in ...]` 遍历 key，可加减 `readonly`/`?`，`as` 重映射 key。组合技：`[K in keyof Animal as `get${Capitalize<K & string>}`]: () => Animal[K]`——从 Animal 形状自动生成 getName/getAge；base 一变派生自动跟，不用手 sync。（`K & string` 因为 keyof 可能含非 string key，Capitalize 要 string。）

**`Record<K,T>`** 就是预包装的 mapped（`{ [P in K]: T }`）：`Record<Status, string>` 不止省字——**强制完备**，漏 `success` key 直接错。心智模型：**系统性整形→mapped**；Pick/Partial/Readonly 都是它造的。

## 4. Conditional types + infer：类型级编程

`T extends U ? X : Y`——编译期按类型关系分支（`IsString<T> = T extends string ? true : false`；NonNullable/Extract/ReturnType 背后都是它）。

**`infer`**：pattern matching 引出命名类型变量——`T extends (...args: any[]) => infer R ? R : never` 掏函数返回；`T extends Array<infer U> ? U : T` 剥一层数组；递归 `FlattenDeep` 剥干净；`T extends [infer F, ...any[]] ? F : never` 拿 tuple 首元。Awaited 拆 Promise 同 pattern。

**大坑 distribution**：被检查的是**裸类型参数**时，conditional 对 union 逐个分发——`ToArray<string|number>` 变成 `string[] | number[]`（逐个算），不是 `(string|number)[]`。过滤/mapping union 时是 feature，想整体当一个看时是 surprise。opt-out：包 tuple——`[T] extends [any] ? ...` 把 union 当整体。

**类型也要单测**：类型不跑，用类型级断言——`Equal<A,B>` 双向 extends + `AssertTrue<T extends true>`——当 utility 的单元测试；mismatch=编译错误指着失败处。

为什么学：标准库 utility 全是这套 machinery 写的。**天天写用不上，读 library typings 必备**；自己写（掏 inner type、过滤 union）是 advanced fluency 的标志。

## 5. Decorators：声明式横切

`@Name` 挂元数据+注入横切行为（logging/validation/路由/DI），不污染业务逻辑。**类定义时跑一次**，不是每实例。

**两套系统必须都认识**（TS 正在过渡）：
- **现代 Stage 3**（TS 5.0+ 默认）：签名 `(value, context)`，typed context（`ClassMethodDecoratorContext`），干净的函数替换模型；
- **Legacy experimental**（`"experimentalDecorators": true`）：签名 `(target, propertyKey, descriptor)`，直接改 descriptor。**NestJS/TypeORM/老 Angular 还要它**——只有 legacy 支持 parameter decorator（DI 框架命根子）；另有 `emitDecoratorMetadata` 给要运行时类型元数据的框架。

各 target mechanics：class decorator 拿构造器（type 写 `{ new (...args: any[]): {} }`，课里强调精确构造器类型别 any），可返回扩展类；method decorator 包函数（legacy：换 `descriptor.value`，`apply` 保 this；现代：`(originalMethod, context)` 返回替换）；property decorator 记字段级元数据（legacy：stash 到构造器；现代：定义时 value 是 undefined，用 field context），不改运行时行为。

**规则**：decorator 加包体积+运行时查找开销，stack trace 难跟；**绝不放核心业务逻辑**——只 infrastructure（logging/validation/authz/metrics）。现实目击：Angular `@Component/@Input/@Injectable`、NestJS `@Controller/@Get/@Post`、class-validator `@IsEmail/@MinLength`。

## 6. JS→TS 迁移：分阶段，不是重写

Playbook：
1. **原地开编译器**：tsconfig `allowJs` + `checkJs`，`noEmit: true`（只 type-check；现代 bundler 如 Vite/esbuild 管 emit——noEmit 开着指给它们源码就行），`strict: true`，`esModuleInterop: true`（CommonJS 默认导入互操作），`lib: ["es2020","dom"]`，`include: ["src"]`。tsc 先当 code reviewer 扫你的 .js。
2. **JSDoc 注解**（`@param`/`@returns`）——真类型信息，编译器当原生 .ts 查，**零文件改名**。
3. 第三方库：DefinitelyTyped（`@types/*`，如 `@types/lodash`）；无类型库→`.d.ts` 里 `declare module "legacy-lib"` 最小 stub 临时桥接（让编译器闭嘴，不等于类型安全）。
4. **逐文件执行**：低风险 utils 先审计→JSDoc→`.js`→`.ts`（`.jsx`→`.tsx`）逐个改名→`any` 换真类型→定 interface/alias→**CI 强制 tsc**，回归合不进去。

为什么这是正道：真实代码库没人 greenfield 重写。**noEmit 是现代 key**：TS 管 type-check，bundler 管 emit，井水不犯河水。每个迁移的文件=技术债变编译器杠杆。

**运行时校验的边界**：静态类型止于编译器——运行时校验是 zod/io-ts/valibot 的活。TS-first 框架：React+TSX、Next.js、NestJS、tRPC。

## 自测 4 问（带答案要点）
1. Pick/Omit/Partial/Required 各造什么 variant？为什么派生？
   → 子集/去字段/全可选/全必填；单一 source of truth 防 drift。
2. `getValue<T, K extends keyof T>(obj: T, key: K): T[K]` 为什么 canonical？
   → 非法 key 编译错+返回类型精确；零 annotation 全类型安全。
3. Distribution 坑？opt-out？
   → 裸类型参数对 union 逐个分发；包 tuple `[T] extends [any]`。
4. 迁移四步？noEmit 的分工？
   → allowJs/checkJs→JSDoc→@types→逐文件+CI；TS 只查类型，bundler emit。
