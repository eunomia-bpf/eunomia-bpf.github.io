# WebAssembly 组件模型与 WIT 格式详解

> 本文部分翻译自 fermyon 的一篇博客，原文地址： [https://www.fermyon.com/blog/webassembly-component-model](https://www.fermyon.com/blog/webassembly-component-model)
>
> 对于一种语言生态而言，标准可能并不是其中最令人激动的部分。但是，“组件模型” 这个看似无聊的名称背后，是Wasm为软件世界带来的最重要的创新。组件模型正在快速发展，并且已经有了参考实现，2023 年很可能是组件模型开始重新定义我们如何编写软件的一年。

WebAssembly 组件模型是一个[提案](https://github.com/WebAssembly/proposals)，旨在通过定义模块在应用程序或库中如何组合来构建核心 WebAssembly 标准。正在开发该模型的[存储库](https://github.com/WebAssembly/component-model)包括非正式规范和相关文档，但是这种详细程度可能会一开始让人有些不知所措。

在本文中，我们首先简要解释组件模型的动因。然后，我们将建立对其工作方式的直觉，并将其与流行操作系统（如Windows、macOS和Linux）上的本地代码链接和运行进行比较。本文的目标受众是已经熟悉本地编译语言（如C、Rust和Go）以及动态链接和进程间通信等概念的开发人员。在下一个部分中，我们讨论一下 eunomia-bpf 社区在组件模型上的一些实践工作，并介绍一种名为WIT（Wasm Interface Type）的文档格式，用于描述组件模型接口的方式。

## **动机**

[WebAssembly规范](https://webassembly.github.io/spec/core/)定义了一种体系结构、平台和语言无关的可执行代码格式。WebAssembly模块可以从主机运行时导入函数、全局变量等，也可以将这些项导出到主机。然而，在运行时没有标准的方式来合并模块，也没有标准的方式来跨模块边界传递高级类型（例如字符串和记录）。

实际上，缺乏模块组合意味着模块必须在构建时静态链接，而不能在运行时动态链接，并且它们不能以标准、便携的方式与其他模块通信。组件模型通过在核心WebAssembly之上提供以下三个主要功能来解决这个问题：

- [接口类型](https://github.com/bytecodealliance/wit-bindgen/blob/main/WIT.md)：一种语言无关的方式，用于基于高级类型（如字符串、记录、集合等）定义模块接口。
- [规范ABI](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md)，它指定高级类型如何以核心WebAssembly的低级类型来表示。
- 模块和组件链接：一种动态组合模块成为组件的机制。这些组件本身可以组合在一起形成更高级的组件。

这些功能共同允许在没有静态链接的重复和安全漏洞的情况下打包和重用代码。这种重用特别适用于多租户云计算，其中成千上万个模块的代码重复会增加昂贵的存储和内存开销。

## **本地代码类比**

如果您熟悉本地代码的链接和执行方式，那么您就会遇到“可执行文件”、“进程”和“共享库”等概念。组件模型具有相似的概念，但使用不同的术语。在这里，我们将为每个本地代码概念匹配其WebAssembly对应物，并考虑它们的相似之处和不同之处。

### **组件 <-> 可执行文件**

Wasm组件就像本地可执行文件一样：惰性且无状态，等待使用。

### **模块 <-> 共享库**

Wasm模块就像共享库（例如`.dll`、`.dylib`或`.so`），可以导出和/或导入符号，以便与其他代码链接在一起。与组件一样，模块是惰性且无状态的，等待使用。

### **组件实例 <-> 进程**

组件的*实例*就像进程；它是组件的加载和运行形式；它是有状态和动态的。像进程一样，几个组件实例可以组织成树，以便每个节点都可以与其直接子节点通信。与进程一样，组件实例之间不共享内存或其他资源，必须通过某种主机介入方式进行通信。

请注意，可以从同一可执行文件运行多个进程-有时同时运行-而这些进程不共享状态。同样，从同一组件产生的实例不共享状态。

### **模块实例 <-> 已加载的共享库**

模块的*实例*就像已加载和链接到进程中的库一样。它是有状态和动态的，与组件的其他部分共享内存和其他资源。

请注意，一个给定的模块可以同时实例化到多个组件中，这些实例之间*不会*共享状态。

一些共享库也可以是可执行文件；即它们可以被链接到进程或作为独立进程执行。同样，Wasm模块实例可以链接到组件实例中或由主机运行时独立运行。

### **Wasm运行时 <-> 操作系统**

Wasm运行时（例如`wasmtime`）类似于操作系统，它负责加载和链接组件（即进程）并在它们之间进行通信。

然而，与大多数提供少量低级进程间通信方法的操作系统不同（例如管道、stdin/stdout等），启用组件的Wasm运行时提供了一种基于[接口类型](https://github.com/bytecodealliance/wit-bindgen/blob/main/WIT.md)和[规范ABI](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md)的单一高级通信方法。此方法类似于诸如gRPC的RPC协议，但它旨在（并优化）用于本地通信，而不是通过网络。

对于跨模块边界的组件内通信，组件模型没有指定标准的ABI。这意味着要在组件内链接的模块必须就适当的ABI达成一致，这可以是特定于语言的或与语言无关的。

与本地模型相反，其中进程间通信通常比库调用更麻烦和复杂，组件模型旨在使组件间和模块间通信都方便和表达。结论是，您可以将Wasm应用程序分成多个单独的沙盒组件，而不需要比共享同一个沙盒的模块更多的工作。

## **注意事项**

以上讨论为简单起见忽略了一些细微差别和例外。例如，流行操作系统上的进程*可以*共享内存、文件句柄等，尽管它们通常不这样做。同样，未来的 WASI API 可能会允许组件实例之间进行类似的共享。

同样，组件可能有除上述 ABI 之外的其他通信方式：网络套接字、文件、JavaScript 环境等。具体细节取决于主机运行时所支持的和授予每个组件的功能。

## **（借用的）可视化**

以下图示托管在[组件模型存储库](https://github.com/WebAssembly/component-model/blob/main/design/mvp/examples/SharedEverythingDynamicLinking.md)，展示了 Wasm 模块和组件的静态和动态链接，右侧的图示表示加载到组件实例中的模块实例。不过，它同样可以展示本地的静态和动态链接，右侧的图示表示进程及其加载的库。

![https://github.com/WebAssembly/component-model/raw/main/design/mvp/examples/images/shared-everything-dynamic-linking.svg](https://github.com/WebAssembly/component-model/raw/main/design/mvp/examples/images/shared-everything-dynamic-linking.svg)

## **结论**

虽然组件模型相对较新且仍在开发中，但它遵循了几十年来用于组织本地代码的相同模式。模块、组件及其实例化方式促进了代码重用和隔离，类似于共享库、可执行文件和进程在您喜爱的操作系统中的使用方式。

此外，组件模型通过提供可跨组件边界使用的表达力强、高级别的ABI，改进了本地模型。这使得在保持代码隔离的同时轻松重用代码，提高了安全性和可靠性。

在Fermyon，我们对组件模型感到兴奋，因为它对于使[Spin](https://spin.fermyon.dev/)微服务安全、高效和易于开发至关重要。Spin已经使用Wasmtime的强大沙盒保证来将服务彼此隔离，并使用wit-bindgen提供高级别、语言无关的API，用于常见功能，如HTTP请求。未来，我们将积极贡献于Wasmtime和其他项目，以帮助实现组件支持，并在它们稳定后采用关键功能用于 Spin。

> 在下一个部分中，我们希望讨论一下 eunomia-bpf 社区在组件模型上面的一些实践工作。eunomia-bpf 社区正在探索 eBPF 和 WebAssembly 相互结合的工具链和运行时: [https://github.com/eunomia-bpf/wasm-bpf](https://github.com/eunomia-bpf/wasm-bpf)  。社区关注于简化 eBPF 程序的编写、分发和动态加载流程，以及探索 eBPF 和 Wasm 相结合的工具链、运行时和运用场景等技术。

## 组件模型的描述文件 - WIT

由于Wasm组件提供了导入和导出“接口”的功能，因此我们可以使用一种称为WIT（Wasm Interface Type）的文档格式来描述这种接口。Wasm接口类型（WIT）格式是一种IDL（接口描述语言），用于以两种主要方式为WebAssembly组件模型提供工具支持：

- WIT是一种开发者友好的格式，用于描述组件的导入和导出。易于阅读和编写，并为从客户语言生成组件以及在主机语言中使用组件提供基础。
- WIT包是在组件生态系统中共享类型和定义的基础。作者可以从其他WIT包导入类型，生成组件，发布代表主机嵌入的WIT包，或协作定义跨平台共享API的WIT定义。

WIT包是WIT文档的集合。每个WIT文档都定义在一个使用文件扩展名`wit`的文件中，例如`foo.wit`，并编码为有效的UTF-8字节。每个WIT文档包含一个`接口`和`世界`的集合。类型可以从包内的同级文档（文件）中导入，还可以通过URL从其他包中导入。

简单来说，WIT文件描述了Wasm组件的导入和导出以及其他相关信息，包括：

- 导入和导出的函数
- 导入和导出的接口
- 用于上述函数和接口的参数或返回值的类型，包括record（类似于结构体）、variant（类似于Rust的enum，一种tagged union）、union（类似于C的union，untagged union）、enum（类似于C的enum，映射到整数）等。

有关类型的更详细信息，请参见[https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md#wit-types](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md#wit-types)

以下是一个WIT文档的示例，其中描述了：

- 一个名为`my-world`的world。可以认为一个world对应一个Wasm组件。
- 组件内导出了一个名为`run`的函数，该函数不接受任何参数，也没有返回值。
- 组件需要导入一个名为`host`的接口，其需要提供一个名为`log`的函数，并接收一个名为`param`的字符串参数。

```wit
default world my-world {
  import host: interface {
    log: func(param: string)
  }
  export run: func()
}
```

## 如何在实际项目中使用 WIT 开发 Wasm 组件

鉴于组件模型和 WIT 已经如此出色，那么我们该如何使用它们呢？

- BytecodeAlliance 开发了 [wit-bindgen](https://github.com/bytecodealliance/wit-bindgen) 和 [wasm-tools](https://github.com/bytecodealliance/wasm-tools) 两个工具，可以帮助我们生成处理接口、字符串、数组、结构体等复杂类型的代码，而我们只需要关心具体函数的实现即可。
- 使用传统的 Wasm 工具链（如 clang、rustc）编译生成 Wasm 模块后，再将其转换成 Wasm 组件。

对于 Rust 开发者来说，Rust 编译器支持原生的 wasm32-wasi 目标，只需要在基于 rustup 的工具链中添加：

```sh
rustup target add wasm32-wasi
```

然后，在 Cargo.toml 文件中添加以下内容，以编译一个 wasi 动态库：

```toml
[lib]
crate-type = ["cdylib"]
```

```sh
cargo add --git https://github.com/bytecodealliance/wit-bindgen wit-bindgen
```

此时，我们需要在与 Cargo.toml 文件相邻的 wit/ 文件夹中添加 WIT 文件。例如，使用 Rust 编写的组件示例代码如下：

```rust
// src/lib.rs

// 使用过程宏为我们在 `host.wit` 中定义的接口生成绑定
wit_bindgen::generate!("host");

// 定义一个自定义类型，并为它实现生成的 `Host` trait，表示为此组件实现了所有必要的导出接口
struct MyHost;

impl Host for MyHost {
    fn run() {
        print("Hello, world!");
    }
}

export_host!(MyHost);
```

通过以上步骤，我们可以使用 Rust 和 WIT 开发 Wasm 组件，方便快捷地实现功能强大的组件模型。

## 示例

### 不同语言编写的 Wasm 组件协同工作

我们在这个例子中展示了通过wit-bindgen分别用C和Rust来编写组件，并且让他们互相调用，然后在wasmtime这个支持组件模型的Wasm运行时上运行。

- [https://github.com/eunomia-bpf/c-rust-component-test](https://github.com/eunomia-bpf/c-rust-component-test)

## btf2wit

一个从BTF（BPF Type Format，一种让eBPF程序在内核中根据不同版本进行重定位的调试信息）生成 WIT 描述文件的小工具，用于使用 Wasm 进行 eBPF 用户态程序的开发

- [https://github.com/eunomia-bpf/btf2wit](https://github.com/eunomia-bpf/btf2wit)

## Rust + wit-bingen

一个用 Rust 编写 eBPF 程序并编译为 Wasm 运行，并使用 btf2wit 生成内核结构体的WIT定义在用户态解析数据的例子：

- [https://github.com/eunomia-bpf/wasm-bpf/tree/main/examples/rust-bootstrap](https://github.com/eunomia-bpf/wasm-bpf/tree/main/examples/rust-bootstrap)

## 关于WIT和组件模型的更多信息

- 截止至 2023 年 3 月，组件模型现在还很不成熟，目前为止没有任何一个完整支持组件模型的Wasm运行时。
  - 但wasmtime现在对组件模型提供初步支持（虽然一堆bug），也因此上面的例子用的是wasmtime。
- 关于WASI。现在已经有了组件模型使用WASI的标准（preview2），但目前为止没有任何运行时实现了这个标准。所以暂时组件模型里用不了WASI。

## 参考资料

- <https://github.com/WebAssembly/component-model> [https://github.com/WebAssembly/component-model](https://github.com/WebAssembly/component-model)
- ****2023 年 WebAssembly 技术五大趋势预测**** [https://zhuanlan.zhihu.com/p/597705400](https://zhuanlan.zhihu.com/p/597705400)
- [webassembly-component-model https://www.fermyon.com/blog/webassembly-component-model](https://www.fermyon.com/blog/webassembly-component-model)
- ****Wasm-bpf: 为云原生 Webassembly 提供通用的 eBPF 内核可编程能力****[https://www.infoq.cn/article/sXrpXdGOTAUW6XWRcKRx](https://www.infoq.cn/article/sXrpXdGOTAUW6XWRcKRx)
