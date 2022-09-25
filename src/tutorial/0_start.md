# eBPF 入门开发实践指南一：介绍与快速上手

<!-- TOC -->

- [1. 什么是eBPF](#1-什么是ebpf)
  - [1.1. 起源](#11-起源)
  - [1.2. 执行逻辑](#12-执行逻辑)
  - [1.3. 架构](#13-架构)
    - [1.3.1. 寄存器设计](#131-寄存器设计)
    - [1.3.2. 指令编码格式](#132-指令编码格式)
  - [1.4. 本节参考文章](#14-本节参考文章)
- [2. 如何使用eBPF编程](#2-如何使用ebpf编程)
  - [2.1. BCC](#21-bcc)
  - [2.2. libbpf-bootstrap](#22-libbpf-bootstrap)
  - [2.3 eunomia-bpf](#23-eunomia-bpf)
  - [2.4 快速上手开发](#24-快速上手开发)

<!-- /TOC -->

## 1. 什么是eBPF

Linux内核一直是实现监控/可观测性、网络和安全功能的理想地方，
但是直接在内核中进行监控并不是一个容易的事情。在传统的Linux软件开发中，
实现这些功能往往都离不开修改内核源码或加载内核模块。修改内核源码是一件非常危险的行为，
稍有不慎可能便会导致系统崩溃，并且每次检验修改的代码都需要重新编译内核，耗时耗力。

加载内核模块虽然来说更为灵活，不需要重新编译源码，但是也可能导致内核崩溃，且随着内核版本的变化
模块也需要进行相应的修改，否则将无法使用。

在这一背景下，eBPF技术应运而生。它是一项革命性技术，能在内核中运行沙箱程序（sandbox programs），而无需修改内核源码或者加载内核模块。用户可以使用其提供的各种接口，实现在内核中追踪、监测系统的作用。

### 1.1. 起源

eBPF的雏形是BPF(Berkeley Packet Filter, 伯克利包过滤器)。BPF于
1992年被Steven McCanne和Van Jacobson在其[论文](https://www.tcpdump.org/papers/bpf-usenix93.pdf)
提出。二人提出BPF的初衷是是提供一种新的数据包过滤方法，该方法的模型如下图所示。   
![](../imgs/original_bpf.png)

相较于其他过滤方法，BPF有两大创新点，首先是它使用了一个新的虚拟机，可以有效地工作在基于寄存器结构的CPU之上。其次是其不会全盘复制数据包的所有信息，只会复制相关数据，可以有效地提高效率。这两大创新使得BPF在实际应用中得到了巨大的成功，在被移植到Linux系统后，其被上层的`libcap`
和`tcpdump`等应用使用，是一个性能卓越的工具。

传统的BPF是32位架构，其指令集编码格式为：

- 16 bit: 操作指令
- 8 bit: 下一条指令跳向正确目标的偏移量
- 8 bit: 下一条指令跳往错误目标的偏移量   

经过十余年的沉积后，2013年，Alexei Starovoitov对BPF进行了彻底地改造，改造后的BPF被命名为eBPF(extended BPF)，于Linux Kernel 3.15中引入Linux内核源码。
eBPF相较于BPF有了革命性的变化。首先在于eBPF支持了更多领域的应用，它不仅支持网络包的过滤，还可以通过
`kprobe`，`tracepoint`,`lsm`等Linux现有的工具对响应事件进行追踪。另一方面，其在使用上也更为
灵活，更为方便。同时，其JIT编译器也得到了升级，解释器也被替换，这直接使得其具有达到平台原生的
执行性能的能力。

### 1.2. 执行逻辑

        eBPF在执行逻辑上和BPF有相似之处，eBPF也可以认为是一个基于寄存器的，使用自定义的64位RISC指令集的
微型"虚拟机"。它可以在Linux内核中，以一种安全可控的方式运行本机编译的eBPF程序并且访问内核函数和内存的子集。
在写好程序后，我们将代码使用llvm编译得到使用BPF指令集的ELF文件，解析出需要注入的部分后调用函数将其
注入内核。用户态的程序和注入内核态中的字节码公用一个位于内核的eBPF Map进行通信，实现数据的传递。同时，
为了防止我们写入的程序本身不会对内核产生较大影响，编译好的字节码在注入内核之前会被eBPF校验器严格地检查。
eBPF程序是由事件驱动的，我们在程序中需要提前确定程序的执行点。编译好的程序被注入内核后，如果提前确定的执行点
被调用，那么注入的程序就会被触发，按照既定方式处理。 

### 1.3. 架构
#### 1.3.1. 寄存器设计

eBPF有11个寄存器，分别是R0~R10，每个寄存器均是64位大小，有相应的32位子寄存器，其指令集是固定的64位宽。

#### 1.3.2. 指令编码格式

eBPF指令编码格式为：

- 8 bit: 存放真实指令码
- 4 bit: 存放指令用到的目标寄存器号
- 4 bit: 存放指令用到的源寄存器号
- 16 bit: 存放偏移量，具体作用取决于指令类型
- 32 bit: 存放立即数

### 1.4. 本节参考文章

[A thorough introduction to eBPF](https://lwn.net/Articles/740157/)
[bpf简介](https://www.collabora.com/news-and-blog/blog/2019/04/05/an-ebpf-overview-part-1-introduction/)
[bpf架构知识](https://www.collabora.com/news-and-blog/blog/2019/04/15/an-ebpf-overview-part-2-machine-and-bytecode/)

## 2. 如何使用eBPF编程

原始的eBPF程序编写是非常繁琐和困难的。为了改变这一现状，
llvm于2015年推出了可以将由高级语言编写的代码编译为eBPF字节码的功能，同时，其将`bpf()`
等原始的系统调用进行了初步地封装，给出了`libbpf`库。这些库会包含将字节码加载到内核中
的函数以及一些其他的关键函数。在Linux的源码包的`samples/bpf/`目录下，有大量Linux
提供的基于`libbpf`的eBPF样例代码。

一个典型的基于`libbpf`的eBPF程序具有`*_kern.c`和`*_user.c`两个文件，
`*_kern.c`中书写在内核中的挂载点以及处理函数，`*_user.c`中书写用户态代码，
完成内核态代码注入以及与用户交互的各种任务。 更为详细的教程可以参考[该视频](https://www.bilibili.com/video/BV1f54y1h74r?spm_id_from=333.999.0.0)
然而由于该方法仍然较难理解且入门存在一定的难度，因此现阶段的eBPF程序开发大多基于一些工具，比如：

- BCC
- BPFtrace
- libbpf-bootstrap

以及还有比较新的工具，例如 `eunomia-bpf` 将 CO-RE eBPF 功能作为服务运行,包含一个工具链和一个运行时，主要功能包括：

- 不需要再为每个 eBPF 工具编写用户态代码框架：大多数情况下只需要编写内核态应用程序，即可实现正确加载运行 eBPF 程序；同时所需编写的内核态代码和 libbpf 完全兼容，可轻松实现迁移；
- 提供基于 async Rust 的 Prometheus 或 OpenTelemetry 自定义可观测性数据收集器，通常仅占用不到1%的资源开销，编写内核态代码和 yaml 配置文件即可实现 eBPF 信息可视化，编译后可在其他机器上通过 API 请求直接部署；

### 2.1. BCC

BCC全称为BPF Compiler Collection，该项目是一个python库，
包含了完整的编写、编译、和加载BPF程序的工具链，以及用于调试和诊断性能问题的工具。

自2015年发布以来，BCC经过上百位贡献者地不断完善后，目前已经包含了大量随时可用的跟踪工具。[其官方项目库](https://github.com/iovisor/bcc/blob/master/docs/tutorial.md)
提供了一个方便上手的教程，用户可以快速地根据教程完成BCC入门工作。

用户可以在BCC上使用Python、Lua等高级语言进行编程。
相较于使用C语言直接编程，这些高级语言具有极大的便捷性，用户只需要使用C来设计内核中的
BPF程序，其余包括编译、解析、加载等工作在内，均可由BCC完成。  

然而使用BCC存在一个缺点便是在于其兼容性并不好。基于BCC的
eBPF程序每次执行时候都需要进行编译，编译则需要用户配置相关的头文件和对应实现。在实际应用中，
相信大家也会有体会，编译依赖问题是一个很棘手的问题。也正是因此，在本项目的开发中我们放弃了BCC，
选择了可以做到一次编译-多次运行的libbpf-bootstrap工具。

### 2.2. libbpf-bootstrap

`libbpf-bootstrap`是一个基于`libbpf`库的BPF开发脚手架，从其
[github](https://github.com/libbpf/libbpf-bootstrap) 上可以得到其源码。

`libbpf-bootstrap`综合了BPF社区过去多年的实践，为开发者提了一个现代化的、便捷的工作流，实
现了一次编译，重复使用的目的。

基于`libbpf-bootstrap`的BPF程序对于源文件有一定的命名规则，
用于生成内核态字节码的bpf文件以`.bpf.c`结尾，用户态加载字节码的文件以`.c`结尾，且这两个文件的
前缀必须相同。  

基于`libbpf-bootstrap`的BPF程序在编译时会先将`*.bpf.c`文件编译为
对应的`.o`文件，然后根据此文件生成`skeleton`文件，即`*.skel.h`，这个文件会包含内核态中定义的一些
数据结构，以及用于装载内核态代码的关键函数。在用户态代码`include`此文件之后调用对应的装载函数即可将
字节码装载到内核中。同样的，`libbpf-bootstrap`也有非常完备的入门教程，用户可以在[该处](https://nakryiko.com/posts/libbpf-bootstrap/)
得到详细的入门操作介绍。

### 2.3 eunomia-bpf

开发、构建和分发 eBPF 一直以来都是一个高门槛的工作，使用 BCC、bpftrace 等工具开发效率高、可移植性好，但是分发部署时需要安装 LLVM、Clang等编译环境，每次运行的时候执行本地或远程编译过程，资源消耗较大；使用原生的 CO-RE libbpf时又需要编写不少用户态加载代码来帮助 eBPF 程序正确加载和从内核中获取上报的信息，同时对于 eBPF 程序的分发、管理也没有很好地解决方案

eunomia-bpf 以一次编译、到处运行的libbpf 为基础实现，是一套编译工具链和运行时，以及一些附加项目，我们希望做到让 eBPF 程序：

- 真正像 JavaScript 或者 WASM 那样易于分发和运行，或者说内核态或可观测性层面的 FaaS：eBPF 即服务；
- 让 eBPF 程序的编译和运行过程大大简化，抛去繁琐的用户态模板编写、繁琐的 BCC 安装流程，只需要编写内核态 eBPF 程序，编译后即可在不同机器上任意内核版本下运行，并且轻松获取可视化结果。

eunomia-bpf 具备的优势

- 大多数情况下只需要编写内核态应用程序，不需要编写任何用户态辅助框架代码；
- 和 libbpf 的原生 C 代码几乎完全兼容，迁移成本极小；
- 仅需一个数十kb的请求，包含预编译 eBPF 程序字节码和辅助信息，即可实现多种完全不同功能的 eBPF 程序的热插拔、热更新；加载时间通常为数十毫秒且内存占用少；（例如使用 [libbpf-bootstrap/bootstrap.bpf.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/bootstrap.bpf.c) ，热加载时间约为 50ms,运行时内存占用约为 5 MB，同时新增更多的 eBPF 程序通常只会增加数百 kB 的内存占用）
- 相比于传统的基于 BCC 或远程编译的分发方式，分发时间和内存占用均减少了一到二个数量级；
- 运行时只需数 MB 且无 llvm、clang 依赖，即可实现一次编译、到处运行；将 eBPF 程序的编译和运行完全解耦，本地预编译好的 eBPF 程序可以直接发送到不同内核版本的远端执行；
- 支持动态分发和加载 tracepoints, fentry, kprobe, lsm 等类型的大多数 eBPF 程序，也支持 ring buffer、perf event 等方式向用户态空间传递信息；
- 提供基于 async Rust 的自定义 Prometheus 或 OpenTelemetry 可观测性数据收集器，通常仅占用不到1%的资源开销；
- 提供 C、Rust 等语言的 SDK，可轻松集成到其他项目中；

eunomia-bpf 的 Github 地址： [https://github.com/eunomia-bpf/eunomia-bpf](https://github.com/eunomia-bpf/eunomia-bpf)

### 2.4 快速上手开发

创建一个新项目：

```sh
$ mkdir hello
$ cd hello
```

在 hello 文件夹中创建一个新的 `hello.bpf.c`，内容如下：

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

typedef int pid_t;

char LICENSE[] SEC("license") = "Dual BSD/GPL";

SEC("tp/syscalls/sys_enter_write")
int handle_tp(void *ctx)
{
 pid_t pid = bpf_get_current_pid_tgid() >> 32;
 bpf_printk("BPF triggered from PID %d.\n", pid);
 return 0;
}
```

假设hello文件夹的父目录是`/path/to/repo`，接下来的步骤：

```console
$ # 下载安装 ecli 二进制
$ wget https://aka.pw/bpf-ecli -O ./ecli && chmod +x ./ecli
$ # 使用容器进行编译，生成一个 package.json 文件，里面是已经编译好的代码和一些辅助信息
$ docker run -it -v /path/to/repo/hello:/src yunwei37/ebpm:latest
$ # 运行 eBPF 程序（root shell）
$ sudo ./ecli run package.json  
```

> 使用 docker 的时候需要把包含 .bpf.c 文件的目录挂载到容器的 /src 目录下，目录中只有一个 .bpf.c 文件；

它会追踪所有进行 write 系统调用的进程的 pid：

```console
$ sudo cat /sys/kernel/debug/tracing/trace_pipe
cat-42755   [003] d...1 48755.529860: bpf_trace_printk: BPF triggered from PID 42755.
             cat-42755   [003] d...1 48755.529874: bpf_trace_printk: BPF triggered from PID 42755.
```

其中 `SEC("tp/syscalls/sys_enter_write")` 告诉了我们需要追踪的系统调用，`bpf_printk` 是 eBPF 程序中的打印函数，`bpf_get_current_pid_tgid` 是获取当前进程的 pid 的函数。

我们编译好的 eBPF 代码同样可以适配多种内核版本，可以直接把 package.json 复制到另外一个机器上，然后不需要重新编译就可以直接运行（CO-RE：Compile Once Run Every Where）；也可以通过网络传输和分发 package.json，通常情况下，压缩后的版本只有几 kb 到几十 kb。

更进一步的文档可以参考：https://eunomia-bpf.github.io/