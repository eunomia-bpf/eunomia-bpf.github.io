# Install

## Remote side

```bash
wget https://github.com/eunomia-bpf/eunomia-bpf/releases/latest/remote-side.tar.gz -O - | tar xvf
mv remote-side/eunomiad /usr/local/bin
mv remote-side/systemd.service /usr/local/systemd/eunomiad.service
systemctl enable eunomiad
systemctl start eunomiad
```

## Dispatch side

```bash
# 下载安装 ecli 二进制
wget https://aka.pw/bpf-ecli -O /usr/local/ecli && chmod +x /usr/local/ecli
# 使用容器进行编译，生成一个 package.json 文件，里面是已经编译好的代码和一些辅助信息
docker run -it -v /path/to/repo:/src yunwei37/ebpm:latest
# 运行 eBPF 程序
cat package.json | sudo ecli run
```

## What's the next?

目前的实现优势：

- 将 eBPF 程序的开发和部署运行解耦；
- 仅需网络的传输延时和 libbpf 重定位、内核验证即可启动 eBPF 程序；
- 通过 HTTP RESTful 接口进行请求，便于插件和二次扩展；
- 仅需一个 eunomia runtime ，体积非常小，在新内核版本上不需要其他依赖；
- 可以更换内核跟踪点或 hook 函数；
- 和内核版本无关；

待改进：

- 目前还不能动态更改 eBPF progs 程序挂载点数量（还需要进一步对 libbpf codegen 进行深度改进），当前可以采用提供一些预编译模板的方式；
- 在开发时需要使用 libbpf-bootstrap 脚手架（可以通过远程编译进一步简化）
- 需要预先实现有用户态的 c 代码模板对 eBPF 程序的输出进行解析（可以通过使用 lua、python 等非预编译脚本语言，随请求发布进行解决）

也许在未来：

- eBPF 程序可以在网页上编译；
- 只需要安装一个小的运行时，或者在应用程序中嵌入一个小的模块（类似 lua 虚拟机一样），有一个通用的 API 接口，再加上一个插件应用商店或者市场，只需要在浏览器中点击一下链接，就能在本地启动一个 eBPF 应用，或者给一个现有的服务接入 eBPF 的超能力，不需要停机重启就能一键更换可观测性插件；
- 每个人都可以通过链接自由地分享自己写好的 eBPF 程序和编译好的 eBPF 代码，甚至可以通过聊天窗口直接发送编译好的程序代码，复制粘贴一下即可运行，不需要任何的编译依赖；

## Eunomia：我们的三个初衷

我们是一个来自浙江大学的学生团队，我们希望开发一个这样的产品：

1. 无需修改代码，无需繁琐的配置，仅需 BTF 和一个微小的二进制即可启动监控和获取 Eunomia 核心功能：

   - 代码无侵入即可开箱即用收集多种指标，仅占用少量内存和 CPU 资源；
   - 告别庞大的镜像和 BCC 编译工具链，最小仅需约 4MB 即可在支持的内核上或容器中启动跟踪；

2. 让 eBPF 程序的分发和使用像网页和 Web 服务一样自然：

   - 数百个节点的集群难以分发和部署 eBPF 程序？

     bpftrace 脚本很方便，但是功能有限？
     Eunomia 支持通过 RESTful API 直接进行本地编译后的 eBPF 代码的分发和热更新，
     仅需约一两百毫秒和几乎可以忽略的 CPU 内存占用即可完成复杂 eBPF 追踪器的部署和更新；

   - 可以通过 HTTP API 高效热插拔 eBPF 追踪器（≈ 100ms），实现按需追踪；

3. 提供一个新手友好的 eBPF 云原生监控框架：

   - 最少仅需继承和修改三四十行代码，
     即可在 Eunomia 中基于 libbpf-bootstrap 脚手架添加
     自定义 eBPF 追踪器、
     匹配安全告警规则、
     获取容器元信息、
     导出数据至 prometheus 和 grafana，
     实现高效的时序数据存储和可视化，轻松体验云原生监控；
   - 提供编译选项，作为一个 C++ 库进行分发；可以嵌入在其他项目中提供服务；
   - 提供了丰富的文档和开发教程，力求降低 eBPF 程序的开发门槛；

我们的项目地址： <https://github.com/yunwei37/Eunomia>

我们的网站： <https://yunwei37.github.io/Eunomia/>

Eunomia 还在概念性探索的阶段，目前支持的功能还很有限，敬请期待

## 参考资料

- <https://jishuin.proginn.com/p/763bfbd73692>
- <https://www.cnblogs.com/davad/p/15891220.html>
- <https://www.infoq.cn/article/3lvxf3zcfy7fp2atq2zo>
- <https://www.infoq.cn/article/iielspcwjf6owd6jmbef>
