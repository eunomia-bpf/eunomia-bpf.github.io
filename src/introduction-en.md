# eunomia-bpf: A dynamic loader to run CO-RE eBPF as a service

[![Actions Status](https://github.com/eunomia-bpf/eunomia-bpf/workflows/Ubuntu/badge.svg)](https://github.com/eunomia-bpf/eunomia-bpf/actions)
[![GitHub release (latest by date)](https://img.shields.io/github/v/release/eunomia-bpf/eunomia-bpf)](https://github.com/eunomia-bpf/eunomia-bpf/releases)
<!-- [![codecov](https://codecov.io/gh/eunomia-bpf/eunomia-bpf/branch/master/graph/badge.svg)](https://codecov.io/gh/filipdutescu/modern-cpp-template) -->

## Our target: Run <abbr title="Compile Once - Run Everywhere">CO-RE</abbr> eBPF function as a service!

- Run `CO-RE` eBPF code without provisioning or managing infrastructure
- simply requests with a json and run `any` pre-compiled ebpf code on `any` kernel version
- very small and simple! Only a binary about `3MB` and no `LLVM/Clang` dependence
- Provide a custom eBPF metrics exporter to `Prometheus` or `OpenTelemetry` in async rust
- as fast as `<100ms` to load and run eBPF program, much more faster than `bcc`'s `ebpf_exporter` when exporting data
- `Distributed` and `decentralized`, No remote compile server needed when loading
- Only write Kernel C code which is compatible with `libbpf`

In general, we develop an approach to compile, transmit, and run most libbpf CO-RE objects with some user space config meta data to help us load and operator the eBPF byte code. The compilation and runtime phases of eBPF is separated completely, so, when loading the eBPF program, only the eBPF byte code and a few kB of meta data is needed.

Most of the time, the only thing you need to do is focus on writing a single eBPF program in the kernel. We have a compiler here: [eunomia-cc](https://github.com/eunomia-bpf/eunomia-cc)
