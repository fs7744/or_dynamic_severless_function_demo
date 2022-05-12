# ebpf 介绍

## ebpf 历史

ebpf 全称是 extended Berkeley Packet Filter, eBPF 技术的前身称为 BPF

在 1992 年 Steven McCanne 和 Van Jacobson 的一篇论文 [《The BSD Packet Filter: A New Architecture for User-level Packet Capture》](https://vodun.org/papers/net-papers/van_jacobson_the_bpf_packet_filter.pdf) 中被第一次被提及.

BPF 诞生的最初目的为更快处理数据包过滤，其在数据包过滤上引入了两大革新：

- 一个新的虚拟机（VM）设计，可以有效地工作在基于寄存器结构的 CPU 之上。

- 应用程序使用缓存只复制与过滤数据包相关的数据，不会复制数据包的所有信息。这样可以最大限度地减少 BPF 处理的数据。

当时 BPF 这种新的技术比最先进的数据包过滤技术快 20 倍。

2014 年初，Alexei Starovoitov 实现了 eBPF。

- 针对现代硬件进行了优化，所以 eBPF 生成的指令集比旧的 BPF 解释器生成的机器码执行得更快。
- 扩展版本也增加了虚拟机中的寄存器数量，将原有的 2 个 32 位寄存器增加到 10 个 64 位寄存器。由于寄存器数量和宽度的增加，开发人员可以使用函数参数自由交换更多的信息，编写更复杂的程序。

总之，这些改进使 eBPF 版本的速度比原来的 BPF 提高了 4 倍。

2014 年 6 月，eBPF[1]扩展到用户空间。这是 BPF 的转折点。正如 Alexei 在提交补丁的注释中写道：“这个补丁展示了 eBPF 的潜力。”

BPF 不再局限于网络栈，已经成为内核顶级的子系统。BPF 程序架构强调安全性和稳定性，看上去更像内核模块，但与内核模块不同，BPF 程序不需要重新编译内核，并且可以确保 BPF 程序运行完成，而不会造成系统的崩溃。

![](/ebpfon.png)

此图展示在过去的 Linux 内核版本中，引入的几个 eBPF 代表性能力，截止 Linux5.14 内核版本，已经拥有了 32 种 eBPF 程序类型。

## eBPF 可以做什么？

从下图可以看到 eBPF 支持的功能：
![](/ebpfpower.png)

虽然 BPF 程序没有明确的分类，但本节根据其主要目的，可以将 BPF 程序分为两类。

### 跟踪

第一类是跟踪。你能编写程序来更好地了解系统正在发生什么。这类程序提供了系统行为及系统硬件的直接信息。同时，它们可以访问特定程序的内存区域，从运行进程中提取执行跟踪信息。另外，它们也可以直接访问为每个特定进程分配的资源，包括文件描述符、CPU 和内存。

代表：

- Falco

  Kubernetes 平台上的一个安全监控项目。

  https://github.com/falcosecurity/falco

- [Sysdig](https://sysdig.com/) 和 [Flowmill](https://www.splunk.com/en_us/investor-relations/acquisitions/flowmill.html)

  两家基于 ebpf 实现观测的商业公司

### 网络

第二类是网络。这类程序可以检测和控制系统的网络流量。它们可以对网络接口的数据包进行过滤，甚至可以完全拒绝数据包。我们可以使用不同的程序类型，将程序附加到内核网络处理的不同阶段上，这样做各有利弊。例如，你可以将 BPF 程序附加到网络驱动程序接收数据包的网络事件上，此时，由于内核没有提供足够的信息，程序也只能访问较少的数据包信息。另一方面，你可以将 BPF 程序附加到数据包传递给用户空间的网络事件。在这种情况下，你可以获得更多的数据包信息，这将有助于你做出更明智的决策，但是完整地处理这些数据包信息需要付出更高的代价。

代表：

- Cilium

  kubernetes 平台上一个完全基于 eBPF 实现数据转发的 CNI 网络插件。

  https://github.com/cilium/cilium

- Katran

  facebook 开源的一个实现四层负载均衡转发的项目。

  https://github.com/facebookincubator/katran

## ebpf 架构

eBPF 程序是在内核中被事件触发的。在一些特定的指令被执行时，这些事件会在 hook 处被捕获。Hook 被触发就会执行 eBPF 程序，对数据进行捕获和操作。
那么 eBPF 是如何工作的呢？

![](/ebpfrun.webp)

![](/ebpfrun2.webp)

简要过程如下：

- 用 C 编写 BPF 程序
- 用 LLVM 将 C 程序编译成对象文件（ELF）
- 用户空间 BPF ELF 加载器（例如 libbpf）解析对象文件
- 加载器通过 bpf() 系统调用将解析后的对象文件注入内核
- 内核验证 BPF 指令，然后对其执行即时编译（JIT），返回程序的一个新文件描述符
- 利用文件描述符 attach 到内核子系统（例如网络子系统）
- 某些子系统还支持将 BPF 程序 offload 到硬件（例如网卡）。

接下来稍微详细解释几个特点

- eBPF 是一种高级虚拟机

  可以在隔离的环境执行代码指令。从某种意义上看，BPF 和 Java 虚拟机（JVM）功能类似，我们可以将高级编程语言编译成机器代码，JVM 是一种运行这种机器代码的专用程序。编译器 LLVM 和 GNU GCC（不久的将来）可提供对 BPF 的支持，将 C 代码编译成 BPF 指令。

- BPF 验证器

  BPF 验证器能阻止可能使内核崩溃的代码。如果代码是安全的，BPF 程序将被加载到内核中。
  主要校验处理流程为：

  - 对程序控制流的深度优先搜索保证 eBPF 能够正常结束

    不会因为任何循环导致内核锁定。严禁使用无法到达的指令；任何包含无法到达的指令的程序都会导致加载失败。

  - 使用校验器模拟执行 eBPF 程序（每次执行一个指令）

    在每次指令执行前后都需要校验虚拟机的状态，保证寄存器和栈的状态都是有效的。严禁越界（代码）跳跃，以及访问越界数据。

  - 校验器会使用 eBPF 程序类型来限制可以从 eBPF 程序调用哪些内核函数，以及访问哪些数据结构

  ![](/ebpfvfier.jpg)

- Linux 内核也为 BPF 指令集成了（JIT）即时编译器。

  在程序被验证后，JIT 编译器会直接将 BPF 字节码转换为机器代码，从而减少运行时的时间开销。该架构具有一个非常有用的特点就是加载 BPF 程序无须重启系统，我们不仅可以在系统启动时通过初始化脚本加载 BPF 程序，也可以按需随时加载程序。

- BPF 映射

  BPF 映射提供内核和用户空间之间双向的数据共享，这意味着我们可以分别从内核和用户空间写入和读取数据。BPF 映射包括一些数据结构类型，从简单数组、哈希映射到自定义的映射，我们甚至可以将整个 BPF 程序保存在 BPF 映射中。

  ![](/bpfmap.webp)

- BPF CO-RE (Compile Once – Run Everywhere)

  [BPF CO-RE](http://arthurchiao.art/blog/bpf-portability-and-co-re-zh/) 需要下列组件之间的紧密合作：

  - BTF 类型信息：使得我们能获取内核、BPF 程序类型及 BPF 代码的关键信息， 这也是下面其他部分的基础；
  - 编译器（clang）：给 BPF C 代码提供了表达能力和记录重定位（relocation）信息的能力；
  - BPF loader (libbpf)：将内核的 BTF 与 BPF 程序联系起来， 将编译之后的 BPF 代码适配到目标机器的特定内核；

  - 内核：虽然对 BPF CO-RE 完全不感知，但提供了一些 BPF 高级特性，使某些高级场景成为可能。

以上几部分相结合，提供了一种开发可移植 BPF 程序的史无前例的能力：这个开发 过程不仅方便（ease），而且具备很强的适配性（adaptability）和表达能力（expressivity）。 在此之前，实现同样的可移植效果只能通过 BCC 在运行时编译 BPF C 程序， BCC 开销非常高。

## 编写 ebpf 的工具

- bcc

  https://github.com/iovisor/bcc

  BCC 是用于创建基于 eBPF 的高效内核跟踪和操作程序的工具包，其中包括一些有用的命令行工具和示例。 BCC 简化了用 C 进行内核检测的 eBPF 程序的编写，包括 LLVM 的包装器以及 Python 和 Lua 的前端。它还提供了用于直接集成到应用程序中的高级库。

- bpftrace

  https://github.com/iovisor/bpftrace

  bpftrace 是 Linux eBPF 的高级跟踪语言。它的语言受 awk 和 C 以及 DTrace 和 SystemTap 等以前的跟踪程序的启发。 bpftrace 使用 LLVM 作为后端将脚本编译为 eBPF 字节码，并利用 BCC 作为与 Linux eBPF 子系统以及现有 Linux 跟踪功能和连接点进行交互的库。

- libbpf

  https://github.com/libbpf/libbpf

  libbpf 作为一个 BPF 程序加载器库，是 BPF CO-RE 的核心组件。

- libbpfgo

  https://github.com/aquasecurity/libbpfgo

  Aqua 实现了对 libbpf C 库的 Go 包装器。

- Dropbox gobpf

  https://github.com/dropbox/goebpf

  Dropbox 支持一小部分程序，但有一个非常干净和方便的用户 API。

- gobpf

  https://github.com/iovisor/gobpf

  IO Visor 的 gobpf 是 BCC 框架的 Go 语言绑定，它更注重于跟踪和性能分析。

- cilium eBPF

  https://github.com/cilium/ebpf

  Cilium 和 Cloudflare 维护一个 纯 Go 语言编写的库 (以下简称 “libbpf-go”)，它将所有 eBPF 系统调用抽象在一个本地 Go 接口后面。

## eBPF 限制

- 一个 BPF 程序的代码数量不能超过 BPF_MAXINSNS (4K)，它的总运行步数不能超过 32K (4.9 内核中这个值改成了 96k)；
- BPF 代码只支持有限循环，这也是为了保证出错时不会出现死循环来 hang 死内核。一个 BPF 程序总的可能的分支数也被限制到 1K；(支持有限循环)
- 为了限制它的作用域，BPF 代码不能访问全局变量，只能访问局部变量。一个 BPF 程序只有 512 字节的堆栈。在开始时会传入一个 ctx 指针，BPF 程序的数据访问就被限制在 ctx 变量和堆栈局部变量中；
- 如果 BPF 需要访问全局变量，它只能访问 BPF map 对象。BPF map 对象是同时能被用户态、BPF 程序、内核态共同访问的，BPF 对 map 的访问通过 helper function 来实现；
- 旧版本 BPF 代码中不支持 BPF 对 BPF 函数的调用，所以所有的 BPF 函数必须声明成 always_inline。在 Linux 内核 4.16 和 LLVM 6.0 以后，才支持 BPF to BPF Calls；
- BPF 虽然不能函数调用，但是它可以使用 Tail Call 机制从一个 BPF 程序直接跳转到另一个 BPF 程序。它需要通过 BPF_MAP_TYPE_PROG_ARRAY 类型的 map 来知道另一个 BPF 程序的指针。这种跳转的次数也是有限制的，32 次(8k 栈空间)
- 内核还可以通过一些额外的手段来加固 BPF 的安全性(Hardening)。主要包括：把 BPF 代码映像和 JIT 代码映像的 page 都锁成只读，JIT 编译时把常量致盲(constant blinding)，以及对 bpf()系统调用的权限限制；
