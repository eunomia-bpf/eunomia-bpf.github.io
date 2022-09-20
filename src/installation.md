# Install

## use binary

To install `ecli`, just download and use the `binary`:

```bash
$ # download the release from https://github.com/eunomia-bpf/eunomia-bpf/releases/latest/download/ecli
$ wget https://aka.pw/bpf-ecli -O ecli && chmod +x ecli
```

You can also found exporter and other tools in [github.com/eunomia-bpf/eunomia-bpf/releases](https://github.com/eunomia-bpf/eunomia-bpf/releases)

## build from source

please refer to：[documents/build.md](https://github.com/eunomia-bpf/eunomia-bpf/blob/master/documents/build.md)

## common problem

if you get a error like `/usr/lib/x86_64-linux-gnu/libstdc++.so.6: version GLIBCXX_3.4.29 not found` on old version kernels,

try:

```console
sudo apt-get upgrade libstdc++6
```

see [https://stackoverflow.com/questions/65349875/where-can-i-find-glibcxx-3-4-29](https://stackoverflow.com/questions/65349875/where-can-i-find-glibcxx-3-4-29)