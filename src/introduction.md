# eunomia-bpf

对比 Web 的发展，eBPF 与内核的关系有点类似于 JavaScript 与浏览器内核的关系。

eunomia-bpf 是一套编译工具链和运行时，以及一些附加项目，我们希望做到让 eBPF 程序真正像 JavaScript 或者 WASM 那样易于分发和运行，或者说内核态或可观测性层面的 FaaS：eBPF 即服务。同时，我们也希望让 eBPF 程序的编译和运行过程大大简化，抛去繁琐的用户态模板编写、繁琐的 BCC 安装流程，只需要编写内核态 eBPF 程序，编译后即可在不同机器上任意内核版本下运行，并且轻松获取可视化结果。

## 核心优势

- 大多数情况下只需要编写内核态应用程序，不需要编写任何用户态辅助框架代码；
- 和 libbpf 的原生 C 代码几乎完全兼容，迁移成本极小；
- 仅需一个数十kb的请求，包含预编译 eBPF 程序字节码和辅助信息，即可实现多种完全不同功能的 eBPF 程序的热插拔、热更新；加载时间通常为数十毫秒且内存占用少；（例如使用 [libbpf-bootstrap/bootstrap.bpf.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/bootstrap.bpf.c) ，热加载时间约为 50ms,运行时内存占用约为 5 MB，同时新增更多的 eBPF 程序通常只会增加数百 kB 的内存占用）
- 相比于传统的基于 BCC 或远程编译的分发方式，分发时间和内存占用均减少了一到二个数量级；
- 运行时只需数 MB 且无 llvm、clang 依赖，即可实现一次编译、到处运行；将 eBPF 程序的编译和运行完全解耦，本地预编译好的 eBPF 程序可以直接发送到不同内核版本的远端执行；
- 支持动态分发和加载 tracepoints, fentry, kprobe, lsm 等类型的大多数 eBPF 程序，也支持 ring buffer、perf event 等方式向用户态空间传递信息；
- 提供基于 async Rust 的自定义 Prometheus 或 OpenTelemetry 可观测性数据收集器，通常仅占用不到1%的资源开销；
- 提供 C、Rust 等语言的 SDK，可轻松集成到其他项目中；

## 概要

我们采用了编译和启动运行 eBPF 程序完全分离的思路：eBPF 程序只需在开发环境编译一遍生成字节码，同时我们的编译工具链会通过源代码分析等手段从内核态的 eBPF 程序中提取少量用于在运行时进行动态加载、动态修改程序的元信息，在真正部署运行时用户态无需任何的本地或远程编译过程即可实现正确加载，和动态调整 eBPF 程序参数。

Eunomia 包含如下几个项目：

- eunomia-bpf：一个基于 libbpf 的 CO-RE eBPF 运行时库，使用 C/C++ 语言。提供 Rust 等语言的 sdk；提供 ecli 作为命令行工具；
- eunomia-cc：一个编译工具链；
- eunomia-exporter：使用 Prometheus 或 OpenTelemetry 进行可观测性数据收集，使用 Rust 编写；
- ebpm-template：使用 Github Action 进行远程编译，本地一键运行；

## 项目地址

- [https://gitee.com/anolis/eunomia](https://gitee.com/anolis/eunomia)
- [https://github.com/eunomia-bpf/eunomia-bpf](https://github.com/eunomia-bpf/eunomia-bpf)
- [https://github.com/eunomia-bpf/eunomia-cc](https://github.com/eunomia-bpf/eunomia-cc)
- [https://github.com/eunomia-bpf/ebpm-template](https://github.com/eunomia-bpf/ebpm-template)