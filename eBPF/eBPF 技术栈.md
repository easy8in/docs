动态插桩：kprobes 和 uprobes

动态插桩（也叫动态跟踪技术），在生产环境中对正在运行的软件插入观测点的能力。

BPF 工具也经常使用动态插桩技术，在内核函数或应用函数的开始或结束位置进行插桩。具体被插桩的函数可以是软件栈中成千上万的运行函数中的任意一个。

Linux 2012 年以 uprobes 形式增加了对用户态函数的动态插桩支持。BPF 跟踪工具既支持 kprobes 又支持 uprobes，因而也就支持对整个软件栈进行动态插桩。

kprobes 技术背景

开发人员在内核或者模块的调试过程中，往往会需要要知道其中的一些函数有无被调用、何时被调用、执行是否正确以及函数的入参和返回值是什么等等。比较简单的做法是在内核代码对应的函数中添加日志打印信息，但这种方式往往需要重新编译内核或模块，重新启动设备之类的，操作较为复杂甚至可能会破坏原有的代码执行过程。

相关参考

[一文看懂eBPF、eBPF的使用（超详细）](https://zhuanlan.zhihu.com/p/480811707)

[Linux内核调试技术——kprobe使用与实现(一)](https://zhuanlan.zhihu.com/p/455693301)

[Linux内核调试技术——kprobe使用与实现(二)](https://zhuanlan.zhihu.com/p/455694175)

[bcc之基于uprobe探测用户态函数实现原理分析](https://zhuanlan.zhihu.com/p/133854805)

[Linux内核调试利器kprobe的使用，从这几点入手！](https://zhuanlan.zhihu.com/p/483206899)

[Debugger 原理揭秘](https://zhuanlan.zhihu.com/p/372135871)

[Kernel Dataplane XDP简介](https://zhuanlan.zhihu.com/p/321387418)

[Calico eBPF Data Plane Of XDP](https://www.tigera.io/learn/guides/ebpf/ebpf-xdp/)

[XDP 实用工具及](https://github.com/xdp-project/xdp-tools)

[XDP 教程](https://github.com/xdp-project/xdp-tutorial)

[XDP 示例](https://github.com/xdp-project/bpf-examples)

[NAT的两种模式SNAT和DNAT介绍](https://www.cnblogs.com/yuhaohao/p/14096431.html)

[Linux 在线源码 一](https://elixir.bootlin.com/linux/latest/source)

[Linux 在线源码 二](https://lxr.missinglinkelectronics.com/)

[基于Netty的四层和七层代理性能方面的一些压力测试](https://zhuanlan.zhihu.com/p/71761881)

[基于Netty 代理网关设计](https://juejin.cn/post/7034715997445554184)

静态插桩： tracepoint 和 USDT（user level statically defined tracing）

动态插桩有一点不好，随着软件版本升级，被插桩的函数有可能被重新命名，或者移除。

另一问题是编译器可能会启用优化，将某些函数做内联（inline）处理，这就使得这些函数无法使用 kprobes 或 uprobes 动态插桩。

对于稳定性和内联问题，有一个统一的解决方案，那就改用静态插桩技术。静态插桩会将稳定的事件名编码到软件代码中，由开发者进行维护。（注解：应该是对常用的被插桩函数，起个事件名，事件名与函数关系由开发者维护）。

BPF 跟踪工具支持内核的静态跟踪点插桩技术，也是指用户态的静态定义跟踪插桩技术。

静态插桩技术美中不足：插桩点会增加开发者的维护成本，因此即使软件中存在静态插桩点，通常数量也十分有限。

开发 BPF 工具，一个推荐的策略是，优先尝试使用静态跟踪技术（跟踪点或 USDT），如果不够的话转而使用动态跟踪技术（ kprobes 或 uprobes ）。

tracepoint 又称为内核跟踪点。

eBPF 开发框架及流程

1. 原始工作流程
	1. 采用 bpf 汇编语言或者变体 C 语言开发
	2. 通过 llvm/clang 编译器编译成 bpf 字节码
	3. 通过 bpf() 函数加载 bpf 字节码到内核中 bpf 虚拟机（JIT）完成代码执行
2. 选用一种前端框架
	1. python 语言可以选择 bcc 框架, bcc 允许用 C 语言来编写 BPF 程序
	2. bpftrace 工具，提供了自己的高级语言
	3. 在底层这两个前端框架都使用了 llvm 中间表示形式和一个 llvm 库来实现 BPF 的编译
	4. 这两个前端框架都是使用了 libbcc 和 libbpf 库
3. libbpf-bootstrap
	1. 应该是整合好的 bpf 开发方案
	2. 完整的、符合社区规范的开发框架

使用 BPF 查看指令集：bpftool

bpftool 在 linux 内核 4.15 版本中添加的。使用 bpftool 可用来查看和操作 BPF 对象，包括 BPF 程序和对应的映射表。它的源代码位于 linux 源代码目录 tools/bpf/bpftool 中。

操作命令可以通过 man bpftool 查看相关文档。

```bash
bpftool prog show    # 查看所有加载到内核的 bpf 程序
					 # 输出内容 第一个数字是 bpf 程序 id
					 # 包含 btf_id ，则可以查看 bpf 程序的调试信息，在大程序内容时展示

bpftool prog dump xlated id ${bpf_id}   # 每个程序内容可以通过 id 打印出来
										# xlated 是模式，将程序翻译成 bpf 汇编的方式

bpftool btf dump id ${bpf_id}  # 查看 btf 调试信息

```

**注：commond not found ，需要安装对应内核版本的 linux-tools-${kernel-version}**

bpf api 查看，头文件 include/uapi/linux/bfp.h
