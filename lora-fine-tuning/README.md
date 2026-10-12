# Fine-Tuning LLMs Using LoRA and QLoRA — 笔记

13 lessons（11 lessons + 2 quizzes 跳过），\~2h，Advanced。拿 Llama 3.1 8B 当 running example：微调 taxonomy → 量化四件套 → LoRA 数学+动手 → QLoRA 三组件+动手，附完整内存梯子实测数。每篇带 4 道自测题，verbose 自包含风格。

- [微调入门](lora-fine-tuning/lora-01-overview.md) —— 为什么微调、transformer 速览、六步 workflow
- [微调 taxonomy](lora-fine-tuning/lora-02-types.md) —— non-parametric vs parametric 全家桶、选型链
- [Quantization](lora-fine-tuning/lora-03-quantization.md) —— PTQ/QAT、四工具、8B 实测 30G→8.5G
- [LoRA](lora-fine-tuning/lora-04-lora.md) —— 低秩数学、三超参、GSM8K 动手
- [QLoRA](lora-fine-tuning/lora-05-qlora.md) —— NF4/双重量化/paged optimizers、5.2G、跨课实操手册
