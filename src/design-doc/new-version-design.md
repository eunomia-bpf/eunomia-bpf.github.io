# 新的版本的尝试

> 基本上把运行时和编译工具链的代码重写了一遍

## 运行时优化：大幅度增强功能性

1. 支持根据配置文件自动生成用户态命令行参数：

比如需要实现一个 ebpf 程序里面的 pid 过滤器，只需要编写内核态代码，在 eBPF 中声明全局变量：

```c
/// @description pid filter for tracing event
const volatile pid_t pid_target = 0;
/// @description uid filter for tracing event
const volatile uid_t uid_target = 0;
/// @description output only failed event
const volatile bool failed = false;
```

(inspired by doxygen)

我们会将注释文档的描述信息提取，放在配置文件里面，并且变成 eBPF 应用的命令行参数：

配置文件信息：

```yaml
....
  data_sections:
  - name: .rodata
    variables:
    - name: pid_target
      type: int
      value: 0
      description: pid filter for tracing event
    - name: uid_target
      type: int
      value: 0
      description: uid filter for tracing event
    - name: failed
      type: bool
      value: false
      description: output only failed event
...
```

使用方式：

```sh
$ sudo ecli run execsnoop.json -p 1234
$ sudo ecli run execsnoop.json -h
Usage: execsnoop [-h] [-v] [-p PID]

    -h, --help            show this help message
    -v, --verbose         show libbpf debug info
    -p PID, --pid_target  pid filter for tracing exec
```

2. 支持自动采集和综合非 ring buffer 和 perf event 的 map，比如 HASH map：

之前使用 ring buffer 和 perf event 的场景会稍微受限，因此需要有一种方法可以自动从 maps 里面采集数据，在源代码里面添加注释即可：

```c
/// @sample {"interval":1000}
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 64);
	__type(key, u32);
	__type(value, struct counter);
} counters SEC(".maps");
```

会自动在配置文件里面体现：

```yaml
....
  maps:
  - ident: counters
    name: counters
    sample:
      interval: 1000
```

就会每隔一秒去采集一次 counters 里面的内容（print_map），以 biopattern 为例：

```sh
TIME       RAND   SEQ    COUNT     KBYTES
22:03:51      0    99      788       3184
22:03:51      0   100        4          0
22:03:51     85    14       21        488
```

3. 输出优化

```
[sudo] password for yunwei: 
TIME     TS      PID     UID     RET     FLAGS   COMM    FNAME   
18:07:18  0      727     0       6       0       irqbalance /proc/interrupts
18:07:18  0      727     0       6       0       irqbalance /proc/stat
18:07:18  0      46693   0       25      524288  ecli    /etc/localtime
18:07:18  0      621     0       7       524288  vmtoolsd /etc/mtab
18:07:18  0      621     0       9       0       vmtoolsd /proc/devices
18:07:18  0      621     0       7       0       vmtoolsd /sys/class/block/sda3/../device/../../../class
18:07:18  0      621     0       -2      0       vmtoolsd /sys/class/block/sda3/../device/../../../label
18:07:18  0      621     0       7       0       vmtoolsd /sys/class/block/sda3/../device/../../../class
18:07:18  0      621     0       -2      0       vmtoolsd /sys/class/block/sda3/../device/../../../label
18:07:18  0      621     0       7       0       vmtoolsd /sys/class/block/sda2/../device/../../../class
18:07:18  0      621     0       -2      0       vmtoolsd /sys/class/block/sda2/../device/../../../label
```

## 编译方面：编译体验优化、格式改进

1. 完全重构了编译工具链和配置文件格式，回归本质的配置文件 + ebpf 字节码 .o 的形式，不强制打包成一个大 JSON（现在编译后生成 xxx.bpf.o 和 xxx.skel.json），对分发使用和人类编辑配置文件更友好，当然也可以编译为 package.json
2. 支持 JSON 和 YAML 两种形式的配置文件（xxx.skel.yaml 和 xxx.skel.json），或者打包成 package.json 和 package.yaml
3. 剔除了一大堆没用的 llvm 信息，尽可能使用 BTF 信息表达符号类型，并且把 BTF 信息隐藏在二进制文件中，让配置文件更可读和可编辑；复用 libbpf 提供的 BTF 处理机制；
4. 支持更多的数据导出类型：enum、struct 等等
5. 编译部分可以不依赖于 docker 运行，可以安装二进制到 ~/.eunomia（对嵌入式或者国内网络更友好，有时也更方便使用），原本 docker 的使用方式还是可以继续使用；
6. 指定源文件和头文件变得更方便，除了和源文件在同一目录的头文件（maps.bpf.h这样的 helper）会被自动包含以外，可以指定任意别的 include 路径、添加任何 clang 编译参数（就和正常调用编译器差不多），文件名没有特定限制，不需要一定是 xxx.bpf.h 和 xxx.bpf.c
7. 把旧的 xxx.bpf.h 修改为 xxx.h，和 libbpf-tools 和 libbpf-boostrap 一致
8. 大幅度优化编译速度和减少编译依赖；全部使用二进制进行编译

新的编译方式：

```console
$ ecc boostrap.bpf.c event.h
Compiling bpf object...
Generating export types...

$ ecc cmd/test/client.bpf.c cmd/test/event.h -p
Compiling bpf object...
Generating export types...
Packing oebpf object and config into client.skel.json...
```

新的配置文件举例：可以直接修改 progs/attach 控制挂载点，variables/value 控制全局变量，maps/data 控制在加载 ebpf 程序时往 map 里面放什么数据，export_types/members 控制往用户态传输什么数据格式，而不需要重新编译。配置文件和 bpf.o 二进制是配套的，应该搭配使用，或者打包成一个 package.json/yaml 分发。打包的时候有压缩，一般来说压缩后的配置文件和二进制合起来的大小在数十 kb 这样。

```yaml
bpf_skel:
  data_sections:
  - name: .rodata
    variables:
    - name: min_duration_ns
      type: unsigned long long
      value: 100
  maps:
  - ident: exec_start
    name: exec_start
    data:
      - key: 123
        value: 456
  - ident: rb
    name: rb
  - ident: rodata
    mmaped: true
    name: client_b.rodata
  obj_name: client_bpf
  progs:
  - attach: tp/sched/sched_process_exec
    link: true
    name: handle_exec
export_types:
- members:
  - name: pid
    type: int
  - name: ppid
    type: int
  - name: comm
    type: char[16]
  - name: filename
    type: char[127]
  - name: exit_event
    type: bool
  name: event
  type_id: 613
```

ecc 更新

```console
$ ecc -h
eunomia-bpf compiler

Usage: ecc [OPTIONS] <SOURCE_PATH> [EXPORT_EVENT_HEADER]

Arguments:
  <SOURCE_PATH>          path of the bpf.c file to compile
  [EXPORT_EVENT_HEADER]  path of the bpf.h header for defining event struct [default: ]

Options:
  -o, --output-path <OUTPUT_PATH>
          path of output bpf object [default: ]
  -a, --additional-cflags <ADDITIONAL_CFLAGS>
          additional c flags for clang [default: ]
  -c, --clang-bin <CLANG_BIN>
          path of clang binary [default: clang]
  -l, --llvm-strip-bin <LLVM_STRIP_BIN>
          path of llvm strip binary [default: llvm-strip]
  -p, --pack-object
          pack bpf object in JSON format with zlib compression and base64 encoding
  -v, --verbose
          print the command execution
  -y, --yaml
          output config skel file in yaml
  -h, --help
          Print help information (use `--help` for more detail)
  -V, --version
          Print version information
```

## examples

添加 uprobe 的 auto attach 例子：

```c
#include <vmlinux.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include "bashreadline.h"

#define TASK_COMM_LEN 16

struct {
	__uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
	__uint(key_size, sizeof(__u32));
	__uint(value_size, sizeof(__u32));
} events SEC(".maps");

SEC("uprobe//bin/bash:readline")
int BPF_KRETPROBE(printret, const void *ret) {
	struct str_t data;
	char comm[TASK_COMM_LEN];
	u32 pid;

	if (!ret)
		return 0;

	bpf_get_current_comm(&comm, sizeof(comm));
	if (comm[0] != 'b' || comm[1] != 'a' || comm[2] != 's' || comm[3] != 'h' || comm[4] != 0 )
		return 0;

	pid = bpf_get_current_pid_tgid() >> 32;
	data.pid = pid;
	bpf_probe_read_user_str(&data.str, sizeof(data.str), ret);

	bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &data, sizeof(data));

	return 0;
};

char LICENSE[] SEC("license") = "GPL";
```

