# eunomia-bpf: A Dynamic Loading Framework for CO-RE eBPF program with WASM

[![Actions Status](https://github.com/eunomia-bpf/eunomia-bpf/workflows/Ubuntu/badge.svg)](https://github.com/eunomia-bpf/eunomia-bpf/actions)
[![GitHub release (latest by date)](https://img.shields.io/github/v/release/eunomia-bpf/eunomia-bpf)](https://github.com/eunomia-bpf/eunomia-bpf/releases)
<!-- [![codecov](https://codecov.io/gh/eunomia-bpf/eunomia-bpf/branch/master/graph/badge.svg)](https://codecov.io/gh/filipdutescu/modern-cpp-template) -->

## Overview

`eunomia-bpf` is a dynamic loading library base on `libbpf`, and a compile toolchain. With eunomia-bpf, you can:

- Write eBPF kernel code only and automatically exposing your data from kernel
- Compile eBPF kernel code to a `JSON`, you can dynamically load it on another machine without recompile
- Compile eBPF program to a `WASM` module, and you can operate the eBPF program or process the data in user space `WASM` runtime
- Package, distribute, and run user-space and kernel-space eBPF programs together in `WASM` module
- very small and simple! The library itself `<1MB` and no `LLVM/Clang` dependence, can be embedded easily in you project
- as fast as `<100ms` and little resource need to dynamically load and run any eBPF program on another kernel version
