# 快速上手

## Github Template

可以使用 Github Template 来快速上手，使用该项目作为模板：[https://github.com/eunomia-bpf/ebpm-template](https://github.com/eunomia-bpf/ebpm-template)

## 在线体验网站

可使用 bolipi 提供的在线体验服务，在线编译，在线运行、在线获取可视化结果：[https://bolipi.com/ebpf/home/online](https://bolipi.com/ebpf/home/online)

## Hello World

新建一个项目，并在其中添加 `hello.bpf.c` 文件：

```sh
$ mkdir hello
$ cd hello
```

在 hello 文件夹中新建一个 hello.bpf.c 的内容如下：

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

假设 hello 文件夹的父目录是 `/path/to/repo`，接下来的步骤：

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

我们编译好的 eBPF 代码同样可以适配多种内核版本，可以直接把 package.json 复制到另外一个机器上，然后不需要重新编译就可以直接运行（CO-RE：Compile Once Run Every Where）；也可以通过网络传输和分发 package.json，通常情况下，压缩后的版本只有几 kb 到几十 kb。

