# 开源之夏课题

## 文件系统

## GPU

## ebpf edge

## embed ebpf vm

## remote memory

## aigc serverless

目标：Combining WrapFS and eBPF to provide a
lightweight Filesystem Sandboxing framework

Run untrusted third-party code from the internet in a safe manner；

Examples:

- Third-party web browser plugins,
- Evaluate a Machine Learning model, etc

File system sandboxing framework

- Unprivileged users and applications
- Fine-grained access control
- Dynamic (programmatic) custom security checks
- Stackable (layered) protection
- Low performance overhead

参考资料：

- https://sandfs.github.io
- “A Lightweight and Fine-grained File System Sandboxing ” Paper
- “when eBPF meets FUSE” in OSS NA’18, LPC’18


### 项目名称

使用 eBPF 实现 GPU 虚拟化

### 项目描述

GPU 是现代计算机中不可或缺的组成部分，其强大的计算能力在很多应用领域发挥着重要作用。然而，GPU 虚拟化一直是一个有挑战的问题。虚拟化 GPU 需要考虑到多个问题，如共享设备、安全隔离和性能损耗等。

本项目旨在利用 eBPF 技术实现一种轻量级的 GPU 虚拟化方案，解决上述问题。使用 eBPF 技术可以在不改变内核的情况下，动态修改系统的行为。eBPF 还提供了强大的安全隔离和性能优化功能，可以用于实现 GPU 虚拟化。

### 所属赛道

2023全国大学生操作系统比赛的“OS功能设计”赛道

### 参赛要求

- 以小组为单位参赛，最多三人一个小组，且小组成员是来自同一所高校的本科生（2023春季学期或之后本科毕业的大一~大四的学生）或研究生
- 如学生参加了多个项目，参赛学生选择一个自己参加的项目参与评奖
- 请遵循“2023全国大学生操作系统比赛”的章程和技术方案要求

### 项目导师

李四（某公司）
* Email: lisi@company.com

### 难度

* 基础特性的难度：高
* 高级特性的难度：较高

### 基本要求

* 使用 eBPF 技术实现 GPU 虚拟化
* 提供安全隔离和共享设备的功能
* 具备低性能损耗

### 预期目标

**安全隔离和共享设备。** GPU 虚拟化需要考虑到安全隔离和共享设备的问题。参赛团队需要利用 eBPF 技术提供一种轻量级的 GPU 虚拟化方案，使得多个应用程序可以安全地共享 GPU 设备，同时实现安全隔离。

**优化 GPU 虚拟化的性能。** GPU 虚拟化可能会对应用程序的性能产生负面影响。参赛团队需要使用 eBPF 技术对 GPU 虚拟化进行性能优化，使得应用程序可以充分发挥 GPU 的计算能力，同时不会受到虚拟化

### 项目名称

Serverless wasm AIGC 平台

### 项目描述

本项目旨在通过使用 wasm 技术、AIGC 引擎和 serverless 平台等技术，探索构建一个高效、可扩展和低成本的人工智能推理平台。该平台可以实现在 serverless 架构下部署和执行 AIGC 模型，并且在执行效率和成本方面有明显的优势。具体来说，参赛团队需要实现以下基本要求：

1. 选择两种或以上的 AIGC 引擎（比如 TensorFlow、PyTorch、MXNet 等）或者自主研发一种新型引擎，并在 serverless 环境中部署和运行这些引擎；
2. 针对特定任务（比如图像处理、自然语言处理等），使用微调技术对模型进行优化，并且在保证模型精度的前提下，尽可能地减少所需的服务器资源和执行时间；
3. 设计和实现一个适用于 serverless 环境的推导框架，能够快速部署和执行 AIGC 模型，并且在计算资源利用和成本方面有较好的表现；
4. 使用 wasm 技术来加速推理和 serverless 的启动，提高系统的执行效率和响应速度。

### 所属赛道

2022 全国大学生操作系统比赛的“OS 功能设计”赛道

### 参赛要求

- 以小组为单位参赛，最多三人一个小组，且小组成员是来自同一所高校的本科生（2022 春季学期或之后本科毕业的大一~大四的学生）或研究生；
- 如学生参加了多个项目，参赛学生选择一个自己参加的项目参与评奖；
- 参赛团队应该熟悉 wasm 技术、AIGC 引擎和 serverless 平台等技术，并且有一定的编程能力和实验室实践经验；
- 参赛团队需要遵守比赛规则和技术要求，并且按照规定的时间节点提交设计方案、报告和演示材料。

### 项目导师

张三（某大学计算机学院）
- Github： https://github.com/zhangsan
- Email： zhangsan@university.edu.cn

### 难度

- 基础特性的难度：中等
- 高级特性的

# AIGC 辅助性能分析工具

### 项目名称

搭建性能分析与 eBPF 的 AIGC 辅助工具

### 项目描述

本项目旨在针对性能分析的痛点，设计并实现一款搭建性能分析与 eBPF 的 AIGC（AI辅助智能编程）工具。该工具可以通过集成各种性能分析工具，例如 perf 等，来进行交互式的性能分析，以及生成 eBPF 代码注入内核，进行更加深入的性能分析。

现有的性能分析工具非常丰富，但是有很多工具不知道该如何选择，也有很多定制化的场景需要针对输入输出进行分析和整理。本项目的目标是通过使用语言模型，从上往下的进行抽象，自动生成性能分析工具的代码。

本项目的参赛者需要具备一定的性能分析知识，同时具备编程能力。建议优先采用 Rust 语言开发。

### 所属赛道

2023 全国大学生操作系统比赛的“OS功能设计”赛道

### 参赛要求

- 以小组为单位参赛，最多三人一个小组，且小组成员是来自同一所高校的本科生（2023春季学期或之后本科毕业的大一~大四的学生）或研究生。
- 如学生参加了多个项目，参赛学生选择一个自己参加的项目参与评奖。
- 请遵循“2023 全国大学生操作系统比赛”的章程和技术方案要求。

### 项目导师

XXX（公司或者高校）
* Github： https://github.com/xxx
* Email： xxx@xxx.com

### 难度

* 基础特性的难度：中等
* 高级特性的难度：较高

### 基本要求

* 集成各种性能分析工具，例如 perf 等。
* 支持 eBPF 代码注入内核进行更加深入的性能分析。
* 建议采用 Rust 语言开发。

### 预期目标

以下是本项目的预期目标，可根据实际情况进行选择。

**自动生成性能分析工具的代码**

在已有的性能分析工具之上，通过使用语言模型从上往下的进行抽象，自动生成性能分析工具的代码。

**可视化展示性能分析结果**

将性能分析结果进行可视化展
