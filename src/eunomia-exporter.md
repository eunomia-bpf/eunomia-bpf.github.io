# ebpf exporter

## 基于 bcc 的 cloudflare/ebpf_exporter

eBPF Exporter 是一个将自定义BPF跟踪数据导出到prometheus的工具，它实现了prometheus获取数据的API，prometheus可以通过这些API主动拉取到自定义的BPF跟踪数据。

具体来说，我们只需要编写一个yaml的配置文件，在配置文件中嵌入BPF代码，运行ebpf_exporter就可以实现导出BPF跟踪数据，而这些数据是可以被prometheus主动拉取到的，进而实现BPF跟踪数据的存储、处理和可视化展示。本文档可用于lmp项目数据采集和实现分布式做参考。

- 基于 bcc，运行 exporter 的时候需要安装配置复杂的 bcc 工具链；
- 需要手动使用配置文件管理 eBPF 程序；
- 和 Prometheus 绑定；

## eunomia-exporter

- 使用 yaml 配置文件来完成自定义 eBPF 指标导出，只需要编写好内核态的 eBPF 代码，编译之后只需要编写一个yaml的配置文件，并加入编译好的 eBPF 字节码信息，运行ebpf_exporter就可以实现导出 BPF 跟踪数据；类似 ebpf_exporter；

- 编译阶段和运行阶段完全分离，可以在本地编译后在服务端上运行，也可以在远程编译后一键在本地运行，这样运行的时候就不需要安装 BCC 环境；

- 可以使用 web API 直接进行 eBPF 程序的热加载、停止和管理，省去了再开发一套 API 进行管理的工作量，可以和后端服务直接对接；也更方便 LMP 插件化的配置手段；

- cloudflare/ebpf_exporter 是基于 bcc 的，需要使用 BCC 的内核态代码，而我们目前还有很多工具是基于 libbpf 的；eunomia-exporter 使用的也是 libbpf 的内核态代码，基于 libbpf 的 eBPF 大部分工具使用 eunomia-exporter 几乎不需要修改内核态代码，即可以完成自定义指标导出的过程，但如果使用 cloudflare/ebpf_exporter 的话，还需要将 libbpf 的内核态 BPF 代码移植回 BCC 写的；

- 使用 rust 的异步运行时编写，二进制体积小、无复杂依赖、性能高（打包以后的镜像可以比 ebpf_exporter 更小）；更轻量级

- 使用了最新的 Opentelemetry SDK 作为 exporter，有非常好的可扩展性，和更完善的可观测性语义支持（Log、metrics、Trace），不仅支持导出到 prometheus （主要适合 metric 类型的信息），也可以导出到其他类型的可观测性组件，例如 Jaeger；

- 只需要写内核态的代码和一些配置信息，和具体用户态的语言生态无关，不需要考虑用户态的接口是 python 还是 go，还是 c；只要底层是 libbpf 的就能用；（go-libbpf、 rust-libbp、C-libbpf），也可以帮助统一输出格式，不需要为每个程序再写一份用户态的加载代码和入口代码；

也许可以使用 eunomia-exporter 对于基于 libbpf 的追踪器进行统一的指标导出，使用 cloudflare/ebpf_exporter 对于基于 BCC 的追踪器进行统一的指标导出？通过统一的 API 和后端对接和管理；

说不定 eunomia-exporter 也可以起到替代许老师之前说的，使用 BCC 的一个统一的命令行接口的效果；

## next

小工具的快速上手？

需要一个编译完成之后的包管理器，让其他人能快速体验 LMP 里面的小工具？

有一个类似于 cargo 的网站，里面有很多编译好的 eBPF 插件，可以直接下载使用，也可以在网页上修改、编译，形成新的小工具；再下载使用；

只需要安装一个 LMP 的 docker 镜像或者二进制（镜像或者二进制很小，只需要几十 MB 或者十几 MB），就可以通过一行命令直接运行使用 LMP 里面的小工具：

比如
```
lmp install opensnoop
```

就能打开浏览器看到可视化的结果，不需要安装什么东西也不需要去使用命令行；

可以不需要配置文件，如果不需要配置文件就是默认执行一个 eBPF 程序，并且默认从 map 里面获取信息

类似于 xmake 和 https://github.com/iovisor/bcc/tree/master/examples/lua 的设计，lua 作为一些可选项，用来帮助完成构建和一些简单的处理

```lua
target("ebpf_program1") -- basic
    add_files("src/opensnoop.bpf.c")

target("ebpf_program1")
    set_kind("uprobe")
    attach_to("handler_entry_uprobe", "lua_pcall") -- attach to lua_pcall in uprobe
    on_event(function (event)
        sort(event)
        os.print("uprobe event: ", event)
        stop("ebpf_program1")
    end)
    add_files("src/uprobe.bpf.c")

entry(function (arg)   -- replace the default entry with 
    if arg == "uprobe" then
        run("ebpf_program1")
        sleep(1000)
        run("ebpf_program2")
    else
        run("ebpf_program1")
    end
end)
```