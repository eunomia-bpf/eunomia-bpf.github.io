# 当 WASM 遇见 eBPF ：使用 WebAssembly 编写、分发、加载运行 eBPF 程序

当今云原生世界中两个最热门的轻量级代码执行沙箱/虚拟机是 eBPF 和 WebAssembly。它们都运行从 C、C++ 和 Rust 等语言编译的高性能字节码程序，并且都是跨平台、可移植的。最大的区别在于： eBPF 在 Linux 内核中运行，而 WebAssembly 在用户空间中运行。我们希望能做一些将二者相互融合的尝试：使用 WASM 来编写通用的 eBPF 程序，然后可以将其分发到任意不同版本、不同架构的 Linux 内核中运行。

## WebAssembly vs. eBPF

eBPF 有一些编程限制，需要经过验证器确保其在内核应用场景中是安全的（例如，没有无限循环、内存越界等），但这也意味着 eBPF 的编程模型不是图灵完备的。相比之下，WebAssembly 是一种图灵完备的语言，具有能够打破沙盒和访问原生 OS 库的扩展，同时 WASM 运行时可以安全地隔离并以接近原生的性能执行用户空间代码。二者的领域主体上有不少差异，但也有不少相互重叠的地方。

有一些在  Linux 内核中运行 WebAssembly 的尝试，然而基本上不太成功。 eBPF 是这个应用场景下更好的选择。但是 WebAssembly 程序可以处理许多类内核的任务，可以被 AOT 编译成原生应用程序。来自 CNCF 的 WasmEdge Runtime 是一个很好的基于 LLVM 的云原生 WebAssembly 编译器。原生应用程序将所有沙箱检查合并到原生库中，这允许 WebAssembly 程序表现得像一个独立的 unikernel “库操作系统”。此外，这种 AOT 编译的沙盒 WebAssembly 应用程序可以在微内核操作系统（如 seL4）上运行，并且可以接管许多“内核级”任务[3]。

虽然 WebAssembly 可以下降到内核级别，但 eBPF 也可以上升到应用程序级别。在 sidecar 代理中，Envoy Proxy 开创了使用 Wasm 作为扩展机制对数据平面进行编程的方法。开发人员可以用 C、C++、Rust、AssemblyScript、Swift 和 TinyGo 等语言编写特定应用的代理逻辑，并将该模块编译到 Wasm 中。通过 proxy-Wasm 标准，代理可以在 Wasmtime 和 WasmEdge 等高性能运行机制中执行那些 Wasm 插件[4]。

尽管有不少应用程序同时使用了二者，但大多数时候这两个虚拟机是相互独立并且没有交集的：例如在可观测性应用中，通过 eBPF 探针获取数据，获取数据之后在用户态引入 WASM 插件模块，进行可配置的数据处理。WASM 模块和 eBPF 程序的分发、运行、加载、控制相互独立，仅仅存在数据流的关联。

一般来说，一个完整的 eBPF 应用程序分为用户空间程序和内核程序两部分：

- 用户空间程序负责加载 BPF 字节码至内核，或负责读取内核回传的统计信息或者事件详情，进行相关的数据处理和控制；
- 内核中的 BPF 字节码负责在内核中执行特定事件，如需要也会将执行的结果通过 maps 或者 perf-event 事件发送至用户空间

用户态程序可以在加载 eBPF 程序前控制一些 eBPF 程序的参数和变量，以及挂载点；也可以通过 map 等等方式进行用户态和内核态之间的双向通信。那么，如果将用户态的所有控制和数据处理逻辑全部移到 WASM 虚拟机中，通过 WASM module 打包和分发 eBPF 字节码，同时在 WASM 虚拟机内部完成整个 eBPF 程序的加载和执行，也许我们就可以将二者的优势结合起来，让任意 eBPF 程序能有如下特性：

- `可移植`：让 eBPF 工具和应用完全平台无关、可移植，不需要进行重新编译即可以跨平台分发；
- `隔离性`：借助 WASM 的可靠性和隔离性，让 eBPF 程序的加载和执行、以及用户态的数据处理流程更加安全可靠；事实上一个 eBPF 应用的用户态控制代码通常远远多于内核态；
- `包管理`：借助 WASM 的生态和工具链，完成 eBPF 程序或工具的分发、管理、加载等工作，目前 eBPF 程序或工具生态可能缺乏一个通用的包管理或插件管理系统；
- `跨语言`：目前 eBPF 程序由多种用户态语言开发（如 Go\Rust\C\C++\Python 等），超过 30 种编程语言可以被编译成 WebAssembly 模块，允许各种背景的开发人员（C、Go、Rust、Java、TypeScript 等）用他们选择的语言编写 eBPF 的用户态程序，而不需要学习新的语言；
- `敏捷性`：对于大型的 eBPF 应用程序，可以使用 WASM 作为插件扩展平台：扩展程序可以在运行时直接从控制平面交付和重新加载。这不仅意味着每个人都可以使用官方和未经修改的应用程序来加载自定义扩展，而且任何 eBPF 程序的错误修复和/或更新都可以在运行时推送和/或测试，而不需要更新和/或重新部署一个新的二进制；
- `轻量级`：WebAssembly 微服务消耗 1% 的资源，与 Linux 容器应用相比，冷启动的时间是 1%：我们也许可以借此实现 eBPF as a service，让 eBPF 程序的加载和执行变得更加轻量级、快速、简便易行；

## 我们的一次尝试

eunomia-bpf 是 `eBPF 技术探索 SIG` 中孵化的项目，目前已经在 [github](https://github.com/eunomia-bpf/eunomia-bpf) 上开源。eunomia-bpf 包含了一个用户态动态加载框架/运行时，以及一个简单的编译工具链。您可以

## 我们是如何做到的

我们的项目架构如下图所示：

![arch](../img/eunomia-arch.png)

总体来说，我们在 WASM 运行时和用户态的 libbpf 中间多加了一层抽象层（eunomia-bpf 库），使得一次编译、到处运行的 eBPF 代码可以从 JSON 对象中动态加载。使用 WASM 或 JSON 编译分发 eBPF 程序的流程图大致如下：

![flow](../img/eunomia-bpf-flow.png)

大致来说，分为三个部分：

1. 用 eunomia-cc 工具链将内核的 eBPF 代码骨架和字节码编译为 JSON 格式
2. 在 WASM 模块中嵌入 JSON 数据，并提供一些 API 用于操作 JSON 形态的 eBPF 程序骨架
3. 从 WASM 模块加载内嵌的 JSON 数据，用 eunomia-bpf 库运行 eBPF 程序骨架。

我们需要完成的仅仅是少量的 native API 和 WASM 运行时的绑定，并且在 wasm 代码中处理 JSON 数据。你可以在一个单一的 `WASM` 模块中拥有多个 `eBPF` 程序。

另外，对于 eunomia-bpf 库而言，不需要 WASM 模块和运行时同样可以启动和动态加载 eBPF 程序，不过此时动态加载运行的就只是内核态的 eBPF 程序字节码。你可以手动或使用任意语言修改 JSON 对象来控制 eBPF 程序的加载和参数，并且通过 eunomia-bpf 自动获取内核态上报的返回数据。对于初学者而言，这可能比使用 WebAssembly 更加简单方便：只需要编写内核态的 eBPF 程序，然后使用 eunomia-cc 工具链将其编译为 JSON 格式，最后使用 eunomia-bpf 库加载和运行即可。完全不用考虑任何用户态的辅助程序，包括 WASM 在内。具体

## 未来的方向

TODO

## 参考资料

1. eunomia-bpf 源代码实现：<https://github.com/eunomia-bpf/eunomia-bpf>
2. eunomia-bpf 龙蜥社区镜像仓库：<https://gitee.com/anolis/eunomia>
3. eBPF 和 WebAssembly：哪种 VM 会制霸云原生时代? <https://juejin.cn/post/7043721713602789407>
4. eBPF 和 Wasm：探索服务网格数据平面的未来: <https://cloudnative.to/blog/ebpf-wasm-service-mesh/>
