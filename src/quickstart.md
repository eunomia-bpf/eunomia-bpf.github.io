# Quick Start

## Github Template

Use this as a github action, to compile online or as an template：[https://github.com/eunomia-bpf/ebpm-template](https://github.com/eunomia-bpf/ebpm-template)

## Online Experience

You can use the online experience service provided by `bolipi`, compile online, run online, and obtain visualization results online：[https://bolipi.com/ebpf/home/online](https://bolipi.com/ebpf/home/online)

## Hello World

Create a new project:

```sh
$ mkdir hello
$ cd hello
```

Create a new `hello.bpf.c` in the hello folder with the following content:

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

Assuming the parent directory of the hello folder is `/path/to/repo`, the next steps:

```console
$ # download ecli binary
$ wget https://aka.pw/bpf-ecli -O ./ecli && chmod +x ./ecli
$ # use docker to compile the ebpf code to a file `package.json`
$ docker run -it -v /path/to/repo/hello:/src yunwei37/ebpm:latest
$ # run eBPF program
$ sudo ./ecli run package.json
```

> When using docker, you need to mount the directory containing the .bpf.c file to the /src directory of the container, and there is only one .bpf.c file in the directory;

It keeps track of the pids of all processes making the `write` system call:

```console
$ sudo cat /sys/kernel/debug/tracing/trace_pipe
cat-42755   [003] d...1 48755.529860: bpf_trace_printk: BPF triggered from PID 42755.
             cat-42755   [003] d...1 48755.529874: bpf_trace_printk: BPF triggered from PID 42755.
```

Our compiled eBPF code can also be adapted to multiple kernel versions. You can directly copy package.json to another machine, and then run it directly without recompiling (CO-RE: Compile Once Run Every Where). `package.json` can be transmitted and distributed over the network, usually, the compressed version is only a few kb to tens of kb.

