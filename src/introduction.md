# eunomia-bpf: A Dynamic Loading Framework for CO-RE eBPF program with WASM

[![Actions Status](https://github.com/eunomia-bpf/eunomia-bpf/workflows/Ubuntu/badge.svg)](https://github.com/eunomia-bpf/eunomia-bpf/actions)
[![GitHub release (latest by date)](https://img.shields.io/github/v/release/eunomia-bpf/eunomia-bpf)](https://github.com/eunomia-bpf/eunomia-bpf/releases)
<!-- [![codecov](https://codecov.io/gh/eunomia-bpf/eunomia-bpf/branch/master/graph/badge.svg)](https://codecov.io/gh/filipdutescu/modern-cpp-template) -->

## Introduction

`eunomia-bpf` is a dynamic loading library base on `CO-RE libbpf`, and a compile toolchain. With eunomia-bpf, you can:

- Write eBPF kernel code only and No code generation, we will automatically exposing your data from kernel
- Compile eBPF kernel code to a `JSON`, you can dynamically load it on another machine without recompile
- Package, distribute, and run user-space and kernel-space eBPF programs together in `OCI` compatible `WASM` module
- very small and simple! The library itself `<1MB` and no `LLVM/Clang` dependence, can be embedded easily in you project
- as fast as `<100ms` and little resource need to dynamically load and run eBPF program

With `eunomia-bpf`, you can also get pre-compiled eBPF programs running from the cloud to the kernel in `1` line of bash, kernel version and architecture independent! For example:

```bash
$ sudo ecli run sigsnoop
```

Base on `eunomia-bpf`, we have an eBPF pacakge manager in [LMP](https://github.com/linuxkerneltravel/lmp) project, with OCI images and [ORAS](https://github.com/oras-project/oras). Powered by WASM, an eBPF program may be able to:

- have isolation and protection for operating system resources, both user-space and kernel-space
- safely execute user-defined or community-contributed eBPF code as plug-ins in a software product
- Write eBPF programs with the language you favor, distribute and run the programs on another kernel or arch.

We have tested on `x86` and `arm` platform, more Architecture tests will be added soon.
