# Wasm 组件模型与模块

## 引言

WebAssembly（简称Wasm）是一种为现代Web应用程序设计的二进制指令格式，它为基于浏览器的应用提供了一种高性能、安全且跨平台的解决方案。Wasm的目标是实现一种在Web平台上快速执行代码的通用方法，从而能够让各种高级编程语言在Web上运行得更快、更安全。

Wasm组件模型是一种以模块为基础的编程范式，它允许开发者将复杂的应用程序拆分成可复用的组件，从而简化开发过程、提高代码的可维护性和可扩展性。在 Wasm 中，组件模型的核心是模块，它封装了代码和数据，以及与其他模块之间的接口。通过引入组件模型，我们可以更好地管理和组织Wasm应用程序，将关注点从单一、庞大的代码库转向模块化、可复用的组件。本文将详细介绍Wasm 组件模型与模块的基本概念、实际应用、生态系统以及未来发展。

### Wasm 的发展背景

随着Web技术的快速发展，越来越多的复杂应用程序需要在浏览器中运行。传统的JavaScript解释器在处理这些复杂的任务时，性能往往不能满足需求。因此，WebAssembly应运而生，它作为一种新的编程模型，能够有效地提高Web应用程序的性能和安全性。

### Wasm 的目标和优势

## Wasm 组件模型概述

### 组件模型的定义

## Wasm 模块

如果写过Wasm程序的话可能会对Wasm模块有一定了解。一个Wasm模块是一个包含了一堆东西的Wasm程序文件，有文本（WAT）和二进制两种形式（两种形式等价，可以互相转换）。

Wasm模块里大概包含：

- 导入声明（声明这个模块需要从外部导入哪些函数之类的东西）
- 导出声明（声明这个模块导出了哪些函数之类的给其他模块，或者host来用）
- 函数的代码

还有一些其他的东西，不过在这里不重要，所以先不说了。

Wasm模块看起来很美好，但其实存在很多问题，比如：

- 函数的返回值/参数不能是浮点数/整数之外的类型。
  - 那如果我们要传递字符串、结构体之类的怎么办呢？
  - 在同一个模块内，编译器可以有一套自己的ABI，只要模块内能工作就可以。
  - 跨模块怎么办？没办法，没有统一的标准。一般我们采取把指针转成整数的方式来传递，但没有什么统一的标准。

- Wasm模块无法定义interface（类似于OOP里的接口）
  - 根据上文我们说的，Wasm模块可以指定这个模块需要导入的函数。
  - 但只有函数是不够的，函数是无状态的，单纯的函数也不便于封装。
  - 如果我们想导入一个实现了一组特定函数的对象怎么办？（interface）

## Wasm组件提供的改进

对于上面这些模块系统的不足，组件系统提出了一些改进方案：

- 提供标准的、通用的、跨平台的接口用以传递复杂的数据类型
- 组件模型给我们提供了interface。Interface指定一组函数签名，Wasm程序可以导出或者导入一个实现了特定interface的对象，使用interface的一方不需要关心interface的具体实现。

- 更具体的改进方案可以见<https://github.com/WebAssembly/component-model/blob/main/design/high-level/UseCases.md>

总而言之，Wasm组件大概要实现这些目标：

- 支持不同语言构建出来组件通过同一种ABI相互调用（可以传字符串、对象之类的东西，而模块只能传整数和浮点数）
- 支持可移植的、语言无关的、安全的对象接口定义
- 进一步增强Wasm的语言独立性、可扩展性、AOT友好性以及与Web平台的集成
- 支持渐进式的改进Wasm组件
- 其他可以见[https://github.com/WebAssembly/component-model/blob/main/design/high-level/Goals.md#component-model-high-level-goals](https://github.com/WebAssembly/component-model/blob/main/design/high-level/Goals.md)

## 组件模型的描述文件 - WIT

既然Wasm组件给我们提供了导入、导出“接口”这种东西的功能，所以自然就有了一种用来描述这种接口的文档格式，被称为[WIT（Wasm Interface Type）](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md)

总的来说，一个WIT文件里描述了一个Wasm组件导入的导出以及一些相关的东西，这些东西包括：

- 导入和导出的函数

- 导入和导出的接口

- 上述函数和接口的参数或返回值所用到的类型，包括record（类似于结构体）、variant（类似于Rust的enum，一种tagged union）、union（类似于C的union，untagged union）、enum（类似于C的enum，映射到整数）等。

关于类型的更详细的信息可以参见[https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md#wit-types](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md)

以下是一个WIT文档的例子，这个文档里描述了：

- 一个名字叫做`my-world`的world。可以认为一个world对应一个Wasm组件
- 组件内导出了一个叫做`run`的函数，这个函数不接受任何参数，也没有返回值
- 组件需要导入一个叫做`host`的interface，其需要提供一个名叫`log`的函数，并接收一个叫做`param`的字符串参数

```plain
default world my-world {
  import host: interface {
    log: func(param: string)
  }

  export run: func()
}
```

## WIT的应用

既然组件模型这么好，又有WIT这么好的东西来做接口描述，那么我们怎么用呢？

- BytecodeAlliance开发了[wit-bindgen](https://github.com/bytecodealliance/wit-bindgen)和[wasm-tools](https://github.com/bytecodealliance/wasm-tools)，可以
- 根据我们写的WIT文件，帮我们生成处理interface、字符串、数组、结构体之类的复杂类型的代码，而我们只需要关心具体函数的实现即可
- 使用传统的Wasm工具链（比如clang、rustc）之类的编译生成Wasm模块后，将其转换成Wasm组件。

比如对于上文所提供的example，将其保存在`test.wit`后，使用`wit-bindgen c test.wit -w test.my-world`即可生成面向C程序的binding，即以下三个文件：

- `my-world.h`，其包含了我们写的组件从外部导入的函数的声明（host接口下的log函数），以及我们需要自己写的函数的声明（run函数）
- `my-world.c`，其中包含了一些工具函数的实现，比如操作组件模型中的字符串类型的工具函数。
- `my_world_component_type.o`包含了几个工具section，用来给`wasm-tools`里的模块转组件的工具用。我们需要把这个文件链接到我们的目标文件里。

`my-world.h`的内容可以参考：

```c
#ifndef __BINDINGS_MY_WORLD_H
#define __BINDINGS_MY_WORLD_H
#ifdef __cplusplus
extern "C" {
#endif

#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <stdbool.h>

typedef struct {
  char*ptr;
  size_t len;
} my_world_string_t;

// Imported Functions from `host`
void host_log(my_world_string_t *param);

// Exported Functions from `my-world`
void my_world_run(void);

// Helper Functions

void my_world_string_set(my_world_string_t *ret, const char*s);
void my_world_string_dup(my_world_string_t *ret, const char*s);
void my_world_string_free(my_world_string_t *ret);

#ifdef __cplusplus
}
#endif
#endif

```

## 一些例子

### 不同组件协同工作

<https://github.com/eunomia-bpf/c-rust-component-test>

我们在这个例子中演示了通过wit-bindgen分别用C和Rust来编写组件，并且让他们互相调用，然后在wasmtime这个支持组件模型的Wasm运行时上跑。

## Rust + wit-bingen

我们做了一个用Rust编写eBPF用户态程序，并使用btf2wit生成内核结构体的WIT定义随后解析数据的例子，具体可以见<https://github.com/eunomia-bpf/wasm-bpf/tree/main/examples/rust-bootstrap>

## 关于WIT和组件模型的更多信息

- 组件模型现在还很不成熟，目前为止[没有任何一个完整支持组件模型的Wasm运行时]( [https://github.com/bytecodealliance/wasm-tools/tree/main/crates/wasm-compose#implementation-status](https://github.com/bytecodealliance/wasm-tools/tree/main/crates/wasm-compose))。
  - 但wasmtime现在对组件模型提供初步支持（虽然一堆bug），也因此上面的例子用的是wasmtime。

- 关于WASI。现在已经有了组件模型使用WASI的标准（[preview2](https://github.com/bytecodealliance/preview2-prototyping)），但目前为止没有任何运行时实现了这个标准。所以暂时组件模型里用不了WASI。

- 关于更多Wasm组件相关的工具。BytecodeAlliance开发了[wasm-tools](https://github.com/bytecodealliance/wasm-tools)，提供了从模块创建组件、合并多个组件等功能。

## eunomia-bpf所做的工作

- <https://github.com/eunomia-bpf/c-rust-component-test>
  - 一个让不同语言写的Wasm组件协同工作的例子

- <https://github.com/eunomia-bpf/btf2wit>
  - 一个从BTF（BPF Type Format，一种让eBPF程序用起来更舒服的调试信息格式）生成WIT的小工具。

## 参考资料

- <https://github.com/WebAssembly/component-model/blob/main/design/high-level/Goals.md>
- <https://github.com/WebAssembly/component-model/blob/main/design/high-level/UseCases.md>
- <https://github.com/WebAssembly/component-model/blob/main/design/high-level/Choices.md>
- <https://github.com/WebAssembly/component-model/blob/main/design/high-level/FAQ.md>
