# 入门（Ch2）：TS 是什么 + tsconfig

> TS = typed superset of JavaScript，编译成纯 JS。**TS 从不改变代码怎么跑，只改变编译器在跑之前知道多少**。

## 1. What is TypeScript

写 index.ts → tsc 编译 → index.js → Node 跑。**类型只存在于编译期，运行时零 enforcement**。

为什么：**把错误从运行时左移到编译期**。JS 里 `double("3")` 静默返回 `"33"`（经典 silent-coercion bug）；TS 里参数标 `number` 就是硬编译错误。`getUser(1).email` 在字段不存在时，TS 编译期拦，JS 运行时 `Cannot read property of undefined` 炸。

两个关键认知：
- TS 的安全是**安全网不是保证**——动态输入+松类型照样漏；
- 默认 **tsc 有类型错误照样 emit JS**——类型错误拦不住 build，除非开 `"noEmitOnError": true`。这让迁移可以增量（一次 port 一个文件），但也意味着**错误只是 advisory，除非 CI 让它 fail build**。

心智模型：**TS 是编译器强制执行的文档**。类型描述意图，编译器让每个调用者遵守。velocity 收益和安全收益一样大：大代码库的 rename 重构和 autocomplete 之所以可信，是类型系统在背后撑着。

## 2. tsconfig：真实项目配置

项目根的 `tsconfig.json` 是编译器控制中心：`compilerOptions`（大脑）、`include`（编哪些，如 "src"）、`exclude`（跳过谁；node_modules 默认 excluded）。

三巨头：
- **target**：emit 哪个版本 JS（如 ES2020，现代常用）；
- **module**：emit 的模块格式（commonjs=Node require/exports，esnext=原生 ES module）；
- **strict**：安全套件总开关。

**strict  bundle**：**strictNullChecks**（null/undefined 不再默默 assign 给一切——缺席必须显式处理）、**noImplicitAny**（推不出来就报错，不许默默 any）、strictFunctionTypes、alwaysStrict。效果：没 annotation 的函数参数直接编译失败。**新项目一律 strict: true**，day one 不开以后 retrofit 疼死。

Trade-off：strict 让重构安全得多，但要更多 upfront annotation。**strict  alone 不 block deploy**——CI 必须 fail on type errors 才有牙。

最小真实配置：`{ compilerOptions: { target, module, strict: true }, include: ["src"], exclude: ["node_modules"] }`。
现代 key insight：**noEmit: true + bundler（Vite/esbuild）负责 emit**——TS 只管 type-check，两者不打架。IDE 里 hover 提示是 baseline inference，**最终真理看 tsc**。

## 自测 4 问（带答案要点）
1. TS 的心智模型？类型在运行时存在吗？
   → 编译器强制执行的文档；只编译期，运行时零 enforcement。
2. tsc 默认有错照样 emit—— implications？
   → 增量迁移友好；但 CI 必须 fail on errors，不然检查没牙。
3. strict 开了哪几个关键检查？各防什么？
   → strictNullChecks（null 乱 assign）、noImplicitAny（默默 any）。
4. noEmit: true 的现代分工？
   → TS 只 type-check，bundler 负责 emit。
