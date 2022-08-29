# Eunomia: 让 eBPF 程序的分发和使用像网页和 Web 服务一样自然

> 我们的项目地址：<https://github.com/yunwei37/Eunomia>

eBPF 是一项革命性的技术，它能在操作系统内核中运行沙箱程序。被用于安全并有效地扩展内核的能力而无需修改内核代码或者加载内核模块。

但是开发、构建和分发 eBPF 一直以来都是一个高门槛的工作，社区先后推出了 BCC、BPFTrace 等前端绑定工作，大大降低了编写和使用 eBPF 技术的门槛，但是这些工具的源码交付方式，需要运行 eBPF 程序的 Linux 系统上安装配套的编译环境，对于分发带来了不少的麻烦，同时将内核适配的问题在运行时才能验证，也不利于提前发现和解决问题。

近年来，为解决不同内核版本中 eBPF 程序的分发和运行问题，社区基于 BTF 技术推出了 CO-RE 功能（“一次编译，到处运行”），一定程度上实现了通过 eBPF 二进制字节码分发，同时也解决了运行在不同内核上的移植问题，但是对于如何打包和分发 eBPF 二进制代码还未有统一简洁的方式。除了 eBPF 程序，目前我们还需要编写用于加载 eBPF 程序和用于读取 eBPF 程序产生数据的各种代码，这往往涉及到通过源码复制粘贴来解决部分问题。

Eunomia 同样基于 CO-RE（Compile Once-Run Everywhere）为基础实现，保留了资源占用低、可移植性强等优点，同时更适合在生产环境批量部署所开发的应用:

Eunomia 想要探索一条全新的路线：在本地编译或者远程服务器编译之后，使用 HTTP RESTful API 直接进行 eBPF 字节码的分发，在生产环境最小仅需 4 MB 的运行时，约 100ms 的时间和微不足道的内存、CPU 占用即可实现 eBPF 代码动态分发、热加载、热更新，不受内核版本限制，不需要在生产环境中安装底层库（如 LLVM、python 等）、搭建环境即可启动；

## eBPF：一个革命性的技术

从古至今，由于内核有监视和控制整个系统的特权，操作系统一直都是实现可观察性、安全性和网络功能的理想场所。同时，操作系统内核也很难进化，因为它的核心角色以及对稳定和安全的高度要求,因此，操作系统级别的创新相比操作系统之外实现的功能较少。

![eBPF.png](img/eBPF.png)

eBPF 从根本上改变了这个定律。通过允许在操作系统内运行沙箱程序，应用开发者能够运行 eBPF 程序在运行时为操作系统增加额外的功能。然后操作系统保证安全和执行效率，就像借助即时编译器（JIT compiler）和验证引擎在本地编译那样。这引发了一波基于 eBPF 的项目，涵盖了一系列广泛的使用案例，例如：

- 在现代数据中心和云原生环境中提供高性能网络和负载均衡；
- 以低开销提取细粒度的安全可观察性数据；
- 帮助应用开发者追踪应用程序；
- 洞悉性能问题和加强容器运行时的安全性

## eBPF 开发和分发方式

众所周知，计算机程序的分发也经历了几个阶段，每个阶段都会带来一次巨大的流量突破和爆发：

1. 使用机器码或汇编编写，和当前机器的架构强绑定，没有移植性可言；
2. 在这之后，伟大的先驱们开发了各种计算机高级编程语言（如 C )和编译器，此时移植需要针对特定的机器指令集架构，有一个编译器实现，并且在移植的时候通过编译器进行源代码的再次编译；
3. 使用虚拟机进行分发和运行（例如 Java），可以预先编译好程序并进行分发，在特定的机器上使用虚拟机进行解译和运行，将编译阶段和运行阶段解耦；
4. 在浏览器中直接浏览 Web 网页或使用 Web 应用程序，仅仅需要一条 url 链接或者说一些 Web 请求，就能在随时随地打开并且运行对应的应用程序，获得相似的体验，不再受设备和操作系统的限制（能打败你的只有网速！）；

同样， eBPF 程序的开发和分发也经历了以下几个阶段：

5. 类似 linux 内核源代码的 samples/bpf 目录下的 eBPF 示例，开发一个匹配当前内核版本的 eBPF 程序，并编译为字节码（或者直接手写字节码），再分发到生产环境中，和当前内核版本强绑定，几乎没有可移植性，开发效率也很低：

```c
int main(int ac, char **argv)
{
    struct bpf_object *obj;
    struct bpf_program *prog;
    int map_fd, prog_fd;
    char filename[256];
    int i, sock, err;
    FILE *f;

    snprintf(filename, sizeof(filename), "%s_kern.o", argv[0]);

    obj = bpf_object__open_file(filename, NULL);
    if (libbpf_get_error(obj))
        return 1;

    prog = bpf_object__next_program(obj, NULL);
    bpf_program__set_type(prog, BPF_PROG_TYPE_SOCKET_FILTER);

    err = bpf_object__load(obj);
    if (err)
        return 1;

    prog_fd = bpf_program__fd(prog);
    map_fd = bpf_object__find_map_fd_by_name(obj, "my_map");
    ...
 }
```

6. 基于 BCC、bpftrace 的开发和分发，一般来说需要把 BCC 和开发工具都安装到容器中，开发效率高、可移植性好，并且支持动态修改内核部分代码，非常灵活。但存在部署依赖 Clang/LLVM 等库，每次运行都要执行 Clang/LLVM 编译，严重消耗 CPU、内存等资源，容易与其它服务争抢的问题；

```python
# initialize BPF
b = BPF(text=bpf_text)
if args.ipv4:
    b.attach_kprobe(event="tcp_v4_connect", fn_name="trace_connect_v4_entry")
    b.attach_kretprobe(event="tcp_v4_connect", fn_name="trace_connect_v4_return")
b.attach_kprobe(event="tcp_close", fn_name="trace_close_entry")
b.attach_kretprobe(event="inet_csk_accept", fn_name="trace_accept_return")

```

7. 基于 libbpf 的 CO-RE（Compile Once-Run Everywhere）技术实现，只要内核已经支持了 BPF 类型格式，就可以使用从内核源码中抽离出来的 libbpf 进行开发，这样可以借助 BTF 获得更好的移植性，但这样依旧需要使用已经编译好的二进制或者 .o 文件进行分发，也需要编写不少辅助函数解决加载 eBPF 程序的问题，分发和开发效率较低；

```c
#include "solisten.skel.h"
...
int main(int argc, char **argv)
{
    ...
    libbpf_set_strict_mode(LIBBPF_STRICT_ALL);
    libbpf_set_print(libbpf_print_fn);

    obj = solisten_bpf__open();
    obj->rodata->target_pid = target_pid;
    err = solisten_bpf__load(obj);
    err = solisten_bpf__attach(obj);
    pb = perf_buffer__new(bpf_map__fd(obj->maps.events), PERF_BUFFER_PAGES,
                  handle_event, handle_lost_events, NULL, NULL);
    ...
}
```

基于 libbpf ，也有其他项目做出了尝试，例如 [coolbpf](https://github.com/aliyun/coolbpf) 项目利用远程编译的思想，把用户的 BPF 程序推送到远端的服务器并返回给用户.o 或.so，提供高级语言如 Python/Rust/Go/C 等进行加载，然后在全量内核版本安全运行；BumbleBee 自动生成与 eBPF 程序相关的用户空间的代码功能，包括加载 eBPF 程序和将 eBPF 程序的数据作为日志、指标和直方图进行展示；但相对而言，使用和分发都还不是很便捷。

## Eunomia: 我们的探索

我们希望能做到：

- 通过 RESTful API，就能以极低的成本分发和部署 eBPF 程序，不需要暂停跟踪、重启；

  ```json
  {
    "name": "...name...",
    "data": "...payload (base64 encoded)...",
    "data_sz": 3088,
    "maps": ["rb"],
    "progs": ["handle_exec"]
  }
  ```

  它就是一个已经编译好的 CO-RE，eBPF 程序，接入 `tracepoint/syscalls/sys_enter_open` 跟踪点，可以观测到所有 open 系统调用文件打开的事件，并且输出它的文件路径、进程的 pid（目前还没有进行压缩，压缩后可以更短）；

- 通过 RESTful API，把 eBPF 程序类似于 Web 服务一样发布，一键完成配置、启动和停止；
- 只需要一个小的运行时就能启动，也可以嵌入到其他应用中，类似 lua 虚拟机一样提供附加的 eBPF 追踪服务；

  对比 BCC 等工具：不存在部署依赖 Clang/LLVM 等库和 每次运行都要执行 Clang/LLVM 编译，严重消耗 CPU、内存等资源等问题；对比于传统的通过镜像分发或通过二进制分发 CO-RE 程序：分发速度和启动速度极快：不需要启动容器、容器调度；不需要进行停机更新和重启，不需要其他的辅助工具、函数或脚本，也不需要任何其他语言的编译运行时环境；

### 快速开始

只需要三个步骤就能完成完整的热更新操作：

1. 在本地通过 eunomia 给定的 eBPF 程序脚手架进行编译，生成对应 eBPF 程序的 字节码；
2. Client 通过 HTTP API 将编译完成的 eBPF 字节码、和对应的元信息直接推送给 eunomia 运行时；
3. Eunomia 运行时直接使用编译好的 eBPF 程序热更新注入内核；

目前版本是这三个步骤，如果结合远程编译的工具，如 [coolbpf](https://github.com/aliyun/coolbpf)，我们也许可以本地只需要点击按钮或者在网页上编写 eBPF 程序，然后一键点击就能完成热更新；

#### 下载安装

可以通过这里下载编译完成的二进制：

<https://github.com/yunwei37/Eunomia/releases/>

我们目前在 ubuntu、fedora 等系统和 5.13、5.15、5.19 等内核版本上进行过测试，需要正确运行的话，应该需要内核的编译选项开启：

```conf
CONFIG_DEBUG_INFO_BTF=y
```

在旧的内核版本上，可能需要额外安装 BTF 文件信息。

我们提供了一个小的 client example，用来测试使用：

<https://github.com/yunwei37/Eunomia/tree/master/bpftools/client-template>

```plaintext
├── client.bpf.c
├── client.cpp
├── client.example1.bpf.c
├── client.example2.bpf.c
├── client.h
├── Makefile
└── Readme.md
```

只需要执行：

```bash
git clone https://github.com/eunomia-bpf/eunomia-bpf
git submodule update --init --recursive
cd bpftools/client-template
make
```

即可生成 client 二进制文件。

#### run

最简单的启动方式（用来测试），把我上面贴的那段 json 复制一下（注意头尾的单引号），放在最后面作为参数，然后就能跑起来啦：

```console
# ./eunomia run hotupdate "{ ... payload ... }"
```

现在我们看看如何使用 HTTP RESTful API：通过以下命令即可启动 Eunomia server：

```console
# eunomia server

start server mode...
[2022-08-18 05:09:32.540] [info] start eunomia...
[2022-08-18 05:09:32.540] [info] start safe module...
[2022-08-18 05:09:32.543] [info] start container manager...
[2022-08-18 05:09:32.895] [info] OS container info: Ubuntu 20.04.4 LTS
[2022-08-18 05:09:33.534] [info] start eBPF tracker...
[2022-08-18 05:09:33.535] [info] process tracker is starting...
```

server 默认监听本地的 8527 地址。使用 client 即可远程分发并启动一个跟踪器：

```console
$ ./client start
200 :["status","ok","id",3]
```

server 会启动一个 eBPF 追踪器，跟踪所有的 open 系统调用，并输出它的文件路径、进程的 pid；

![files](img/files.png)

使用 list 就能查看所有的已经在运行的检查器：

```console
$ ./client list
200 :["status","ok","list",[[1,"process"],[2,"files"],[3,"tcpconnect"],[4,"hotupdate"]]]
```

把刚刚启动的那个检查器停下来：

```console
$ ./client stop 4
200 :["status","ok"]
```

server 端就会收到信号并且停止当前在运行的 id 为 4 的 eBPF 程序：

```bash
[2022-08-18 05:24:18.830] [info] accept http request to stop tracker
[2022-08-18 05:24:18.831] [info] waiting for the tracker to cleanup...
[2022-08-18 05:24:18.879] [info] updatable tracker exited.
[2022-08-18 05:24:19.832] [info] exit!
```

我们提供了三个示例：

- bpftools/client-template/client.bpf.c: 文件打开
- bpftools/client-template/client.example1.bpf.c: 进程启动
- bpftools/client-template/client.example2.bpf.c: 进程停止

现在直接把 client.example1.bpf.c 的内容复制到 client.bpf.c 中，然后 make 重新编译，不需要停下 server，再重新发起请求：

```bash
$ ./client start
200 :["status","ok","id",4]
```

就可以看到，eunomia server 已经在进行进程启动的追踪，输出 pid、ppid、进程可执行文件路径等信息：

![proccess](img/process.png)

> 我们目前还在测试阶段，API 还未完善，可能有所疏漏，也还会有不少的变化和更新）

### 代码

<https://github.com/yunwei37/Eunomia/blob/master/bpftools/client-template/client.bpf.c>

eBPF 的代码和正常的 libbpf 追踪器代码没有什么不同：

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>
#include "client.h"

char LICENSE[] SEC("license") = "Dual BSD/GPL";

struct {
 __uint(type, BPF_MAP_TYPE_RINGBUF);
 __uint(max_entries, 256 * 1024);
} rb SEC(".maps");

SEC("tracepoint/syscalls/sys_enter_open")
int handle_exec(struct trace_event_raw_sys_enter* ctx)
{
 u64 id = bpf_get_current_pid_tgid();

 u32 pid = id;
 struct event *e;
 /* reserve sample from BPF ringbuf */
 e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
 if (!e)
  return 0;
 e->pid = pid;
 bpf_probe_read_str(&e->filename, sizeof(e->filename), (void *)ctx->args[0]);
 /* successfully submit it to user-space for post-processing */
 bpf_ringbuf_submit(e, 0);
 return 0;
}
```

<https://github.com/yunwei37/Eunomia/blob/master/bpftools/client-template/client.cpp>

client 的代码也非常简单（~70 行） ，我们只需要看这个部分，它将 eBPF 程序和元信息解析编码后生成了 json 请求，并且通过网络发送：

```http
POST /start HTTP/1.1
Host: example.com
Content-Type: application/json

{
  "name": "hotupdate",
  "export_handlers": [
    {
      "name": "plain_text",
      "args": []
    }
  ],
  "args": []
}
```

在 <https://github.com/yunwei37/Eunomia/blob/master/bpftools/hot-update/update_tracker.h> 中，我们有一个预先加载好的模板进行解析，并交由 libbpf 进行重定位和 eBPF 热加载：

```c
  /* Load and verify BPF application */
  struct eBPF_update_meta_data eBPF_data;
  eBPF_data.from_json_str(json_str);
  skel = single_prog_update_bpf__decode_open(eBPF_data);
  if (!skel)
  {
    fprintf(stderr, "Failed to open and load BPF skeleton\n");
    return 1;
  }

  /* Load & verify BPF programs */
  err = update_bpf__load(skel);
  ...
```
