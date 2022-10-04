# eunomia-bpf: A Dynamic Loading Framework for CO-RE eBPF program with WASM

[![Actions Status](https://github.com/eunomia-bpf/eunomia-bpf/workflows/Ubuntu/badge.svg)](https://github.com/eunomia-bpf/eunomia-bpf/actions)
[![GitHub release (latest by date)](https://img.shields.io/github/v/release/eunomia-bpf/eunomia-bpf)](https://github.com/eunomia-bpf/eunomia-bpf/releases)
<!-- [![codecov](https://codecov.io/gh/eunomia-bpf/eunomia-bpf/branch/master/graph/badge.svg)](https://codecov.io/gh/filipdutescu/modern-cpp-template) -->

## Our target 

With eunomia-bpf, you can:

- Run `CO-RE` eBPF code without provisioning or managing infrastructure
- Write eBPF kernel code only and compile it to a `JSON`, you can dynamically load it on another machine without recompile
- Compile eBPF program to a `WASM` module, and you can operate the eBPF program or process the data in user space `WASM` runtime
- very small and simple! The library itself `<1MB` and no `LLVM/Clang` dependence, can be embedded easily in you project
- as fast as `<100ms` to dynamically load and run any eBPF program, much more faster than `bcc`

In general, we develop an approach to compile, transmit, and run most libbpf CO-RE objects with some user space config meta data to help you load and operator the eBPF byte code. The compilation and runtime phases of eBPF is separated completely, so, when loading the eBPF program, only the eBPF byte code and a few kB of meta data is needed.

Most of the time, the only thing you need to do is focus on writing a single eBPF program in the kernel. If you want to have a user space program to operate the eBPF program, you can write a `WASM` module to do it.
