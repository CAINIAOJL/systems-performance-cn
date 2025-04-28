# Chapter 3: 操作系统

了解作系统及其内核对于系统性能分析至关重要。您经常需要开发并测试有关系统行为的假设，例如系统调用的执行方式、内核如何在 CPU 上调度线程、有限的内存如何影响性能或文件系统如何处理 I/O。这些活动将要求您应用您对作系统和内核的了解。

本章的学习目标是：
■ 学习内核术语：上下文切换、交换、分页、抢占等。
■ 了解内核和系统调用的角色。
■ 获得内核内部结构的应用知识，包括：中断、调度程序、虚拟内存和 I/O 堆栈。
■ 了解如何将内核性能特性从 Unix 添加到 Linux。
■ 对扩展 BPF 有基本的了解

本章概述了作系统和内核，并假定对本书的其余部分有所了解。如果您错过了作系统课程，您可以将其视为速成课程。留意你知识中的任何空白，因为最后会有一次考试（我在开玩笑，这只是一个测验）。有关内核内部的更多信息，请参阅本章末尾的参考资料。

本章分为三个部分： 
■ 术语 列出了基本术语。
■ 背景 总结了关键的作系统和内核概念。
■ 内核总结了 Linux 和其他内核的实现细节

与性能相关的领域，包括 CPU 调度、内存、磁盘、文件系统、网络和许多特定的性能工具，将在以下章节中更详细地介绍

# 3.1 术语
作为参考，以下是本书中使用的核心作系统术语。其中许多概念也将在本章和后面的章节中更详细地解释。
■ 操作系统：这是指安装在系统上以便它可以启动和执行程序的软件和文件。它包括内核、管理工具和系统库。
■ 内核：内核是管理系统的程序，包括 （取决于内核模型） 硬件设备、内存和 CPU 调度。它在允许直接访问硬件的特权 CPU 模式下运行，称为内核模式。
■ 进程：用于执行程序的作系统抽象和环境。该程序在用户模式下运行，可以通过系统调用或内核陷阱访问内核模式（例如，用于执行设备 I/O）。
■ 线程：可调度在 CPU 上运行的可执行上下文。内核具有多个线程，并且一个进程包含一个或多个线程。■ 任务：一个 Linux 可运行实体，可以指一个进程（具有单个线程）、来自多线程进程的线程或内核线程
■ BPF 程序：在 BPF1 执行环境中运行的内核模式程序。
■ 主内存：系统的物理内存（例如 RAM）。
■ 虚拟内存：支持多任务和超额订阅的主内存抽象。实际上，它是一种无限的资源。
■ 内核空间：内核的虚拟内存地址空间。
■ 用户空间：进程的虚拟内存地址空间。
■ 用户领域：用户级程序和库（/usr/bin、/usr/lib...）。
■ 上下文切换：从运行一个线程或进程到另一个线程或进程的切换。这是内核 CPU 调度程序的正常功能，涉及将正在运行的 CPU 寄存器集（线程上下文）切换到新的集合。
■ 模式切换：内核模式和用户模式之间的切换。
■ 系统调用 （syscall）：一种定义明确的协议，供用户程序请求内核执行特权作，包括设备 I/O。
■ 处理器：不要与进程混淆，处理器是包含一个或多个 CPU 的物理芯片。
■ Trap：发送到内核以请求系统例程（特权作）的信号。陷阱类型包括系统调用、处理器异常和中断。
■ 硬件中断：物理设备发送到内核的信号，通常用于请求 I/O 服务。中断是一种陷阱。
术语表包括更多术语供本章参考，包括地址空间、缓冲区、CPU、文件描述符、POSIX 和寄存器。

# 3.2 背景
以下各节介绍通用作系统和内核概念，并将帮助您了解任何作系统。为了帮助您理解，本节包括一些 Linux 实现细节。接下来的 3.3 内核和 3.4 Linux 将重点介绍 Unix、BSD 和 Linux 内核实现的细节

# #3.2.1 内核
内核是作系统的核心软件。它的作用取决于内核模型：包括 Linux 和 BSD 在内的类 Unix作系统具有一个整体内核，用于管理 CPU 调度、内存、文件系统、网络协议和系统设备（磁盘、网络接口等）。该内核模型如图 3.1 所示。

还显示了系统库，与单独的系统调用相比，它们通常用于提供更丰富、更轻松的程序 ming 接口。应用程序包括所有正在运行的用户级软件，包括数据库、Web 服务器、管理工具和作系统 shell。

系统库在这里被描绘成一个虚线环，以表明应用程序可以直接调用系统调用（syscalls）。例如，Golang 运行时有自己的 syscall 层，不需要系统库 libc。传统上，此图是用完整的环绘制的，这反映了从中心的内核开始的特权级别降低（该模型起源于 Multics [Graham 68]，Unix 的前身）。

还存在其他内核模型：微内核采用一个小内核，其功能已移至用户模式程序;而 UniKernels 将 kernel 和 application code 一起编译为单个程序。还有一些混合内核，例如 Windows NT 内核，它们同时使用整体内核和微内核的方法。部分 3.5， 其他主题中总结了这些内容。

Linux 最近改变了它的模型，允许使用一种新的软件类型：扩展 BPF，它支持安全的内核模式应用程序以及它自己的内核 API：BPF 帮助程序。这允许用 BPF 重写某些应用程序和系统功能，从而提供更高级别的安全性和性能。如图 3.2 所示。

扩展 BPF 总结为第 3.4.4 节 扩展 BPF。

# 内核执行
内核是一个大型程序，通常有数百万行代码。它主要在用户级程序进行系统调用或设备发送中断时按需执行。一些内核线程异步运行以进行内务处理，其中可能包括内核时钟例程和内存管理任务，但这些任务试图轻量级并且消耗很少的 CPU 资源

执行频繁 I/O 的工作负载（如 Web 服务器）主要在内核上下文中执行。计算密集型工作负载通常在用户模式下运行，不受内核中断。人们可能很容易认为内核不会影响这些计算密集型工作负载的性能，但在许多情况下它确实如此。最明显的是 CPU 争用，当其他线程争夺 CPU 资源时，内核调度程序需要决定哪个将运行，哪些将等待。内核还选择线程将在哪个 CPU 上运行，并且可以为进程选择具有更暖硬件缓存或更好内存位置的 CPU，以显著提高性能。

# #3.2.2 内核和用户模式
内核在一种称为内核模式的特殊 CPU 模式下运行，允许对设备进行完全访问和执行特权指令。内核仲裁设备访问以支持多任务处理，除非明确允许，否则可以防止进程和用户访问彼此的数据。

用户程序（进程）在用户模式下运行，它们通过系统调用（例如 I/O）向内核请求特权作。

内核和用户模式在处理器上使用特权环（或保护环）实现，如图 3.1 中的模型所示。例如，x86 处理器支持四个权限环，编号为 0 到 3。通常只使用两个或三个：用于用户模式、内核模式和虚拟机管理程序（如果存在）。仅在内核模式下允许访问设备的特权指令;在用户模式下执行它们会导致异常，然后由内核处理（例如，生成权限被拒绝错误）

在传统内核中，系统调用是通过切换到内核模式，然后执行系统调用代码来执行的。如图 3.3 所示。

在用户模式和内核模式之间切换是一种模式切换。

所有系统调用模式切换。一些系统调用还会进行上下文切换：那些阻塞的调用（例如磁盘和网络 I/O）将进行上下文切换，以便另一个线程可以在第一个线程被阻塞时运行。

由于模式和上下文切换会消耗少量开销（CPU 周期）3，因此有各种优化措施来避免它们，包括：

■ 用户模式 syscalls：可以单独在用户模式库中实现一些 syscall。Linux 内核通过导出映射到进程地址空间的虚拟动态共享对象 （vDSO） 来实现此目的，该空间包含诸如 gettimeofday（2） 和 getcpu（2） [Drysdale 14] 之类的系统调用。
■ 内存映射：用于需求分页（参见第 7 章 内存， 第 7.2.3 节 需求分页），它也可以用于数据存储和其他 I/O，避免系统调用开销
■ 内核旁路：这允许用户模式程序直接访问设备，绕过系统调用和典型的内核代码路径。例如，用于网络的 DPDK：数据平面开发工具包。
■ 内核模式应用程序：这些应用程序包括在内核中实现的 TUX Web 服务器 [Lever 00]，以及最近的扩展 BPF 技术，如图 3.2 所示。

内核和用户模式有自己的软件执行上下文，包括堆栈和注册表。某些处理器体系结构（例如 SPARC）为内核使用单独的地址空间，这意味着模式切换也必须更改虚拟内存上下文。

# #3.2.3 系统调用
系统调用请求内核执行特权系统例程。有数百个可用的系统调用，但内核维护者做出了一些努力，以保持该数量尽可能小，以保持内核简单（Unix 哲学;[汤普森 78]）。更复杂的接口可以在用户空间作为系统库构建，从而更容易开发和维护。作系统通常包括一个 C 标准库，该库为许多常见的系统调用（例如，libc 或 glibc 库）提供了易于使用的接口。

表 3.1 中列出了需要记住的关键系统调用

系统调用都有详细的文档记录，每个调用都有一个手册页，该手册页通常随运行系统一起提供。它们还具有通常简单一致的界面，并在需要时使用错误代码来描述错误（例如，ENOENT 表示“没有此类文件或目录”）。

其中许多系统调用都有明显的目的。以下是一些可能不太明显的常见用法：

■ ioctl（2）：这通常用于从内核请求其他作，特别是对于系统管理工具，当另一个（更明显的）系统调用不合适时。请参阅下面的示例。
■ mmap（2）：这通常用于将可执行文件和库映射到进程地址空间，以及用于内存映射文件。它有时用于分配进程的工作内存，而不是基于 brk（2） 的 malloc（2），以降低系统调用速率并提高性能（由于涉及的权衡：内存映射管理，这并不总是有效）。
■ brk（2）：用于扩展堆指针，它定义了进程的工作内存大小。当堆中的现有空间无法满足 malloc（3） （内存分配） 调用时，它通常由系统内存分配库执行。参见第 7 章 “内存”。
■ futex（2）：这个系统调用用于处理用户空间锁的一部分：可能被阻塞的部分。

如果不熟悉系统调用，可以在其手册页中了解更多信息（这些内容位于手册页的第 2 节：syscalls）。

ioctl（2） 系统调用可能是最难学习的，因为它的歧义性。作为其用法的一个示例，Linux perf（1） 工具（在第 6 章 CPU 中介绍）执行特权作来协调性能检测。不是为每个作添加系统调用，而是添加单个系统调用：perf_event_open（2），它返回用于 ioctl（2） 的文件描述符。然后可以使用不同的参数调用此 ioctl（2） 来执行不同的所需作。例如，ioctl（fd， PERF_EVENT_IOC_ENABLE） 启用检测。在此示例中PERF_EVENT_IOC_ENABLE，开发人员可以更轻松地添加和更改参数。

# #3.2.4 中断
中断是向处理器发出的信号，表明发生了需要处理的事件，并中断处理器的当前执行以处理该事件。它通常会导致处理器进入内核模式（如果尚未进入），保存当前线程状态，然后运行中断服务程序 （ISR） 来处理事件。

有外部硬件生成的异步中断和软件指令生成的同步中断。这些如图 3.4 所示。

为简单起见，图 3.4 显示了发送到内核进行处理的所有中断;这些首先发送到 CPU，CPU 在内核中选择 ISR 来运行事件

# 异步中断
硬件设备可以向处理器发送中断服务请求 （IRQ），这些请求以异步方式到达当前正在运行的软件。硬件中断的示例包括： 
■ 指示磁盘 I/O 完成的磁盘设备 
■ 指示故障情况的硬件 
■ 指示数据包到达的网络接口 
■ 输入设备：键盘和鼠标输入

为了解释异步中断的概念，图 3.5 中显示了一个示例场景，显示了在 CPU 0 上运行的数据库 （MySQL） 从文件系统中读取数据时的时间流逝。必须从磁盘获取文件系统内容，因此在数据库等待时，调度程序上下文会切换到另一个线程（Java 应用程序）。一段时间后，磁盘 I/O 完成，

但此时数据库不再在 CPU 0 上运行。完成中断与数据库异步发生，如图 3.5 中的虚线所示。

# 同步中断
同步中断由软件指令生成。下面使用术语 traps、exceptions 和 faults 描述了不同类型的软件中断;但是，这些术语通常可以互换使用。

■ Traps：对内核的故意调用，例如通过 int（中断）指令。syscall 的一种实现涉及使用 syscall 处理程序的向量调用 int 指令（例如，Linux x86 上的 int 0x80 ）。int 引发软件中断。
■ Exceptions：异常情况，例如执行除以零的指令。
■ Faults：通常用于内存事件的术语，例如，在没有 MMU 映射的情况下访问内存位置触发的页面错误。请参见第 7 章 “内存”。

对于这些中断，负责的软件和指令仍在 CPU 上

# 中断线程
中断服务程序 （ISR） 旨在尽快运行，以减少中断活动线程的影响。如果中断需要执行更多的工作，特别是如果它可能在锁上阻塞，它可以由内核调度的中断线程处理。如图 3.6 所示。

如何实现取决于内核版本。在 Linux 上，设备驱动程序可以修改为两半，上半部分快速处理中断，并将工作安排到下半部分以便稍后处理 [Corbet 05]。快速处理中断非常重要，因为上半部分在中断禁用模式下运行，以推迟新中断的交付，如果运行时间过长，可能会导致其他线程出现延迟问题。下半部分可以是 tasklet 或工作队列;后者是可以由 kernel 调度的线程，并且可以在必要时休眠。

例如，Linux 网络驱动程序的上半部分用于处理入站数据包的 IRQ，它调用下半部分将数据包推送到网络堆栈中。下半部分作为 softirq （软件中断） 实现。

从中断到达到服务的时间是中断延迟，这与硬件和实现有关。这是实时或低延迟系统的研究主题。

# 中断掩码
内核中的某些代码路径无法安全地中断。例如，内核代码在系统调用期间获取旋转锁，因为 interrupt 可能也需要旋转锁。在持有此类锁的情况下进行中断可能会导致死锁。为了防止这种情况，内核可以通过设置 CPU 的 interrupt mask register来临时屏蔽中断。中断禁用时间应尽可能短，因为它可能会干扰被其他中断唤醒的应用程序的及时执行。对于实时系统（具有严格的响应时间要求）来说，这是一个重要因素。中断禁用时间也是性能分析的目标（Ftrace irqsoff 跟踪器直接支持这种分析，如第 14 章 Ftrace 中提到）。

一些高优先级事件不应被忽略，因此被实现为不可屏蔽的 inter rupts （NMI）。例如，Linux 可以使用智能平台管理接口 （IPMI）看门狗计时器，用于检查内核是否根据一段时间内缺少中断而似乎已锁定。如果是这样，则监视程序可以发出 NMI 中断以重新启动系统5。

# #3.2.5 clock 和 Idle
原始 Unix 内核的核心组件是 clock（） 例程，从 timer inter rupt 执行。它历来以每秒 60、100 或 1,000 次6（通常以赫兹表示：每秒周期数）执行，每次执行称为一个滴答声7。其功能包括更新系统时间、使计时器和时间片过期以进行线程调度、维护 CPU 统计信息以及执行调度的内核例程。
时钟存在性能问题，在以后的内核中得到了改进，包括：

■ 滴答延迟：对于 100 赫兹时钟，定时器可能会遇到高达 10 毫秒的额外延迟，因为它等待在下一个滴答时处理。此问题已使用高分辨率实时中断得到修复，以便立即执行。
■ Tick 开销：Tick 会占用 CPU 周期并略微扰乱应用程序，并且是导致作系统抖动的原因之一。现代处理器还具有动态电源功能，可以在空闲期间关闭部件。clock 例程会中断这个空闲时间，这可能会不必要地消耗功率。

现代 kernels 已将许多功能从 clock 例程转移到按需 inter rupts，以努力创建一个无 tick 的 kernel。这允许处理器更长时间地保持睡眠状态，从而减少开销并提高电源效率。

Linux clock 例程是 scheduler_tick（） ，Linux 有办法在没有任何 CPU 负载时省略调用 clock。时钟本身通常以 250 赫兹（由 CONFIG_HZ Kconfig 选项和变体配置），并且它的调用由 NO_HZ 功能（由 CONFIG_NO_HZ 和变体配置）减少，该功能现在普遍启用 [Linux 20a]。

# 空闲线程
当 CPU 没有要执行的工作时，内核会安排一个等待工作的占位符线程，称为空闲线程。一个简单的实现将在循环中检查新工作的可用性。在现代 Linux 中，空闲任务可以调用 hlt （halt） 指令来关闭 CPU 电源，直到收到下一个中断，从而节省电量

# #3.2.6 进程
进程是执行用户级程序的环境。它由内存地址空间、文件描述符、线程堆栈和寄存器组成。在某些方面，进程就像一台虚拟的早期计算机，其中只有一个程序使用自己的寄存器和堆栈执行。进程由内核进行多任务处理，通常支持在单个系统上执行数千个进程。它们由其进程 ID （PID） 单独标识，PID 是一个唯一的数字标识符。

进程包含一个或多个线程，这些线程在进程地址空间中运行并共享相同的文件描述符。线程是由堆栈、寄存器和指令指针（也称为程序计数器）组成的可执行上下文。多个线程允许单个进程跨多个 CPU 并行执行。在 Linux 上，线程和进程都是任务。

内核启动的第一个进程称为 “init”，来自 /sbin/init（默认），PID 为 1，用于启动用户空间服务。在 Unix 中，这涉及从 /etc 运行启动脚本，这种方法现在称为 SysV（在 Unix System V 之后）。Linux 发行版现在通常使用 systemd 软件来启动服务并跟踪其依赖项。

# 进程创建
在 Unix 系统上，通常使用 fork（2） 系统调用创建进程。在 Linux 上，C 库通常通过包装通用的 clone（2） 系统调用来实现 fork 函数。这些系统调用会创建进程的副本，并带有自己的进程 ID。然后可以调用 exec（2） 系统调用（或变体，例如 execve（2））来开始执行不同的程序。

图 3.7 显示了执行 ls 命令的 bash shell （bash） 的进程创建示例。

fork（2） 或 clone（2） 系统调用可以使用写入时复制 （COW） 策略来提高性能。这将添加对前一个地址空间的引用，而不是复制所有内容。一旦任一进程修改了多引用内存，就会为修改创建单独的副本。此策略推迟或消除了复制内存的需要，从而减少内存和 CPU 使用率。

# 流程生命周期
流程的生命周期如图 3.8 所示。这是一个简化的图表;对于现代多线程作系统，线程被调度和运行，并且还有一些关于它们如何映射到进程状态的额外实现细节（有关更详细的图表，请参见第 5 章中的图 5.6 和 5.7）。

on-proc 状态用于在处理器 （CPU） 上运行。准备运行状态是指进程可运行，但正在等待 CPU 运行队列轮到 CPU 启动时。大多数 I/O 将阻塞，使进程处于睡眠状态，直到 I/O 完成并唤醒进程。僵停状态发生在进程终止期间，当进程等待直到其进程状态被父进程获取或直到它被内核删除时。

# 进程环境
过程环境如图 3.9 所示;它由进程地址空间中的数据和内核中的元数据（上下文）组成。

内核上下文由各种进程属性和统计信息组成：其进程 ID （PID）、所有者的用户 ID （UID） 和各种时间。这些通常通过 ps（1） 和 top（1） 命令进行检查。它还具有一组文件描述符，这些描述符引用打开的文件，并且（通常）在线程之间共享。

此示例描绘了两个线程，每个线程都包含一些元数据，包括内核 context8 中的优先级和用户地址空间中的用户堆栈。该图不是按比例绘制的;与进程地址空间相比，内核上下文非常小。

用户地址空间包含进程的内存段：可执行文件、库和堆。有关更多详细信息，请参见第 7 章 “内存”。

在 Linux 上，每个线程都有自己的用户堆栈和一个内核异常堆栈9 [Owens 20]

# #3.2.7 栈
堆栈是临时数据的内存存储区域，组织为后进先出 （LIFO） 列表。它用于存储比适合 CPU register set 的数据更重要的数据。调用 func tion 时，返回地址将保存到堆栈中。如果在调用后需要它们的值，也可以将一些寄存器保存到堆栈中。当被调用的函数完成时，它会恢复任何需要的寄存器，并通过从堆栈中获取返回地址，将执行传递给调用函数。堆栈还可用于将参数传递给函数。堆栈上与函数执行相关的数据集称为堆栈帧。

通过检查线程堆栈中所有堆栈帧中保存的返回地址（称为堆栈遍历的过程），可以查看当前正在执行的函数的调用路径。此调用路径称为堆栈回溯跟踪或堆栈跟踪。在性能工程中，它通常简称为“堆栈”。这些堆栈可以回答执行某项作的原因，并且是调试和性能分析的宝贵工具。

# 如何读取堆栈
以下示例内核堆栈（来自 Linux）显示了 TCP 传输所采用的路径，由跟踪工具打印
 tcp_sendmsg+1 ----------叶
    sock_sendmsg+62 --------根
    SYSC_sendto+319
    sys_sendto+14
    do_syscall_64+115
    entry_SYSCALL_64_after_hwframe+61
堆栈通常按叶到根的顺序打印，因此打印的第一行是当前正在执行的函数，其下面是它的父级，然后是它的祖父级，依此类推。在此示例中，正在执行 tcp_ sendmsg（） 函数，该函数由 sock_sendmsg（） 调用。在此堆栈示例中，函数名称的右侧是指令偏移量，显示函数中的位置。第一行显示 tcp_sendmsg（） 偏移量 1（这将是第二条指令），由 sendmsg（） 偏移量 62 调用sock_。仅当您希望对所采用的代码路径进行低级理解时，此偏移量才有用，直到指令级别。

通过向下读取堆栈，可以看到完整的祖先：function、parent、grandparent，等等。或者，通过自下而上阅读，您可以遵循当前函数的执行路径：我们是如何到达这里的。

由于堆栈公开了通过源代码采用的内部路径，因此除了代码本身之外，通常没有这些函数的文档。对于此示例堆栈，这是 Linux 内核源代码。一个例外是函数是 API 的一部分并被记录在案。

# 用户和内核堆栈
在执行系统调用时，进程线程有两个堆栈：用户级堆栈和内核级堆栈。他们的范围如图 3.10 所示。

在系统调用期间，阻塞线程的用户级堆栈不会更改，因为线程在内核上下文中执行时使用的是单独的内核级堆栈。（信号处理程序可能是一个例外，它可能会根据其配置借用用户级堆栈。

在 Linux 上，有多个内核堆栈用于不同的目的。系统调用使用与每个线程关联的内核异常堆栈，并且还有与软硬内部协议 （IRQ） 关联的堆栈 [Bovet 05]。

# #3.2.8 虚拟内存
虚拟内存是主内存的抽象，为进程和内核提供自己的、几乎无限的 主内存私有视图。它支持多任务处理，允许进程和内核在自己的私有地址空间上运行，而无需担心争用。它还支持主内存的超额订阅，允许作系统根据需要在主内存和辅助存储（磁盘）之间透明地映射虚拟内存。

虚拟内存是通过处理器和作系统的支持实现的。它不是实际内存，大多数作系统仅在首次填充（写入）内存时按需将虚拟内存映射到实际内存

有关虚拟内存的更多信息，请参见第 7 章 “内存”。

# 内存管理
虚拟内存允许使用辅助存储扩展主内存，而内核则努力将最活跃的数据保留在主内存中。有两种内核方案：

■ 进程交换在主内存和辅助存储之间移动整个进程。
■ 分页移动称为页面的小内存单位（例如，4 KB）。

进程交换是原始的 Unix 方法，可能会导致严重的性能损失。分页效率更高，并且随着分页虚拟内存的引入而被添加到 BSD 中。在这两种情况下，最近最少使用（或最近未使用的）内存将移至二级存储器，并仅在再次需要时移回主存储器。

在 Linux 中，术语 swapping 用于指代分页。Linux 内核不支持整个线程和进程的（较旧的）Unix 样式进程交换。

有关 paging 和 swap 的更多信息，请参见第 7 章 内存。

# #3.2.9 调度
Unix 及其衍生产品是分时系统，通过在它们之间划分执行时间，允许多个进程同时运行。处理器和单个 CPU 上的进程调度由调度程序执行，调度程序是作系统内核的一个关键组件。调度器的作用如图 3.12 所示，它显示了调度器在线程（在 Linux 中是任务）上运行，将它们映射到 CPU。

基本目的是在活动进程和线程之间分配 CPU 时间，并保持优先级的概念，以便可以更快地执行更重要的工作。调度器跟踪所有处于准备运行状态的线程，传统上是在称为运行队列的按优先级队列上 [Bach 86]。现代内核可以为每个 CPU 实现这些队列，也可以使用除队列之外的其他数据结构来跟踪线程。当想要运行的线程数超过可用 CPU 数时，优先级较低的线程将等待轮到它们。大多数内核线程的运行优先级高于用户级进程。

调度程序可以动态修改进程优先级，以提高某些工作负载的性能。工作负载可分为

■ CPU 绑定：执行大量计算的应用程序，例如科学和数学数学分析，预计运行时间较长（秒、分钟、小时、天，甚至更长）。这些资源会受到 CPU 资源的限制。
■ I/O 绑定：执行 I/O 的应用程序，计算量很少，例如 Web 服务器、文件服务器和交互式 shell，这些应用程序需要低延迟响应。当它们的负载增加时，它们会受到存储或网络资源的 I/O 限制。

可追溯到 UNIX 的常用调度策略可识别 CPU 密集型工作负载并降低其优先级，从而允许 I/O 密集型工作负载（更需要低延迟响应）更快地运行。这可以通过计算最近的计算时间（在 CPU 上执行的时间）与实际时间（经过的时间）的比率，并降低具有高（计算）比率的进程的优先级来实现 [Thompson 78]。此机制优先选择运行时间较短的进程，这些进程通常是执行 I/O 的进程，包括人工交互进程。

现代内核支持多个调度类或调度策略 （Linux），它们应用不同的算法来管理优先级和可运行的线程。这些可能包括实时调度，它使用高于所有非关键工作（包括内核线程）的优先级。除了抢占支持（稍后介绍）之外，实时调度还为需要它的系统提供可预测的低延迟调度

有关内核调度程序和其他调度算法的更多信息，请参见第 6 章 “CPU”

# #3.2.10 文件系统
文件系统是将数据组织为文件和目录。它们有一个基于文件的接口来访问它们，通常基于 POSIX 标准。内核支持多种文件系统类型和实例。提供文件系统是作系统最重要的角色之一，曾经被描述为最重要的角色 [Ritchie 74]。

作系统提供了一个全局文件命名空间，该命名空间组织为从根级别 （“/”） 开始的自上而下的树拓扑。文件系统通过挂载、将自己的树附加到目录（挂载点）来加入树。这允许最终用户横向导航文件命名空间，而不管底层文件系统类型如何

典型的作系统可以如图 3.13 所示进行组织

顶级目录包括 etc （用于系统配置文件）、usr （用于系统提供的用户级程序和库）、dev （用于设备节点）、var （用于各种文件（包括系统日志）、tmp （用于临时文件）和 home （用于用户主目录）。在图片的示例中， var 和 home 可能驻留在它们自己的文件系统实例和单独的存储设备上;但是，可以像树的任何其他组件一样访问它们。

大多数文件系统类型使用存储设备（磁盘）来存储其内容。某些文件系统类型是由内核动态创建的，例如 /proc 和 /dev。

内核通常提供不同的方法将进程隔离到文件命名空间的一部分，包括 chroot（8），在 Linux 上，挂载命名空间，通常用于容器（参见第 11 章 云计算）。

# VFS
虚拟文件系统 （VFS） 是用于抽象文件系统类型的内核接口，最初由 Sun Microsystems 开发，以便 Unix 文件系统 （UFS） 和网络文件系统 （NFS） 可以更轻松地共存。它的作用如图 3.14 所示

VFS 接口可以更轻松地向内核添加新的文件系统类型。它还支持提供全局文件命名空间，如前所述，以便用户程序和应用程序可以透明地访问各种文件系统类型

# I/O 堆栈
对于基于存储设备的文件系统，从用户级软件到存储设备的路径称为 I/O 堆栈。这是前面显示的整个软件堆栈的子集。通用 I/O 堆栈如图 3.15 所示。

图 3.15 显示了绕过文件系统在左侧阻止设备的直接路径。此路径有时由管理工具和数据库使用。

第 8 章 “文件系统”中详细介绍了文件系统及其性能，第 9 章 “磁盘”中介绍了构建文件系统的存储设备

# #3.2.11缓存
由于磁盘 I/O 历来具有较高的延迟，因此软件堆栈的许多层都试图通过缓存读取和缓冲写入来避免它。缓存可能包括表 3.2 中所示的缓存（按检查顺序）。

例如，缓冲区缓存是主内存的一个区域，用于存储最近使用的磁盘块。如果存在请求的块，则可以立即从缓存中提供磁盘读取，从而避免磁盘 I/O 的高延迟。

存在的缓存类型因系统和环境而异。

# #3.2.12 网络
现代内核提供了一堆内置网络协议，允许系统通过网络进行通信并参与分布式系统环境。这称为网络堆栈或 TCP/IP 堆栈，以常用的 TCP 和 IP 协议命名。用户级应用程序通过称为套接字的可编程端点访问网络。

连接到网络的物理设备是网络接口，通常在网络接口卡 （NIC） 上提供。系统管理员的历史职责是将 IP 地址与网络接口相关联，以便它可以与网络通信;现在，这些映射通常通过动态主机配置协议 （DHCP） 自动执行。

网络协议不会经常更改，但有一种新的传输协议正在被广泛采用：QUIC（在第 10 章 网络中总结）。协议增强功能和选项更改得更频繁，例如更新的 TCP 选项和 TCP 拥塞控制算法。较新的协议和增强功能通常需要内核支持（用户空间协议实现除外）。另一个变化是支持不同的网络接口卡，这需要内核的新设备驱动程序。

有关网络和网络性能的更多信息，请参见第 10 章 “网络”。

# #3.2.13 设备驱动
内核必须与各种物理设备通信。这种通信是通过设备驱动程序实现的：用于设备管理和 I/O 的内核软件。设备驱动程序通常由开发硬件设备的供应商提供。某些内核支持可插拔设备驱动程序，这些驱动程序可以在不需要重新启动系统的情况下加载和卸载。

设备驱动程序可以为其设备提供字符和/或块接口。字符设备（也称为原始设备）提供对任何 I/O 大小的无缓冲顺序访问，具体到单个字符，具体取决于设备。此类设备包括键盘和串行端口（在原始 Unix 中，还包括纸带和行式打印机设备）。

块设备以块为单位执行 I/O，每个块以前为 512 字节。这些可以根据它们的区块偏移量随机访问，该偏移量从区块开始时的 0 开始装置。在最初的 Unix 中，块设备接口还在称为缓冲区缓存的主内存区域中提供块设备缓冲区的缓存以提高性能。在 Linux 中，此缓冲区缓存现在是页面缓存的一部分。

# #3.2.14 多处理器
多处理器支持允许作系统使用多个 CPU 实例来并行执行工作。它通常实现为对称多处理 （SMP），其中所有 CPU 都得到同等对待。这在技术上很难实现，给在并行运行的线程之间访问和共享内存和 CPU 带来了问题。在多处理器系统上，也可能有主内存组连接到非一致性内存访问 （NUMA） 架构中的不同插槽（物理处理器），这也带来了性能挑战。有关内存访问和体系结构的详细信息，请参见第 6 章 “CPU”，以及第 7 章 “内存”

# IPIs
对于多处理器系统，有时 CPU 需要协调，例如内存转换条目的缓存一致性（通知其他 CPU 某个条目（如果缓存了，则现在已过时）。CPU 可以使用处理器间中断 （IPI）（也称为 SMP 调用或 CPU 交叉调用）请求其他 CPU 或所有 CPU 立即执行此类工作。IPI 是处理器中断，旨在快速执行，以最大限度地减少其他线程的中断。

IPI 也可以通过抢占来使用。

# #3.2.15 先买权
内核抢占支持允许高优先级用户级线程中断内核并执行。这使得实时系统能够在给定的时间限制内执行工作，包括飞机和医疗设备使用的系统。支持抢占的内核被称为完全抢占式，尽管实际上它仍然会有一些无法中断的小关键代码路径。

Linux 支持的另一种方法是自愿内核抢占，其中内核代码中的逻辑停止点可以检查并执行抢占。这避免了支持完全抢占式内核的一些复杂性，并为常见工作负载提供了低延迟抢占。自愿内核抢占通常在 Linux 中通过 CONFIG_PREEMPT_ VOLUNTARY Kconfig 选项启用;还有CONFIG_PREEMPT允许所有内核代码（关键部分除外）都是抢占的，CONFIG_PREEMPT_NONE禁用抢占，以更高的延迟为代价提高吞吐量。

# #3.2.16 资源管理
作系统可以提供各种可配置的控件，用于微调对系统资源（如 CPU、内存、磁盘和网络）的访问。这些是资源控制，可用于管理运行不同应用程序或托管多个租户（云计算）的系统的性能。此类控制可以为每个进程（或进程组）施加固定的资源使用限制，或者采用更灵活的方法，允许在它们之间共享备用使用。早期版本的 Unix 和 BSD 具有基本的每进程资源控制，包括 nice（1） 的 CPU 优先级，以及 ulimit（1） 的一些资源限制。

对于 Linux，已开发控制组 （cgroups） 并将其集成到 Linux 2.6.24 （2008） 中，并且从那时起添加了各种附加控件。这些记录在 Documentation/cgroups 下的内核源代码中。还有一个改进的统一分层方案，称为 cgroup v2，在 Linux 4.5 （2016） 中提供，并记录在 Documentation/admin guide/cgroup-v2.rst 中。

具体的资源控制将在后面的章节中适当地提及。第 11 章 云计算 中介绍了一个示例用例，用于管理作系统虚拟化租户的性能。

# #3.2.17 观测性
作系统由内核、库和程序组成。这些程序包括用于观察系统活动和分析性能的工具，通常安装在 /usr/bin 和 /usr/ sbin 中。第三方工具也可能安装在系统上，以提供额外的可观察性。第 4 章介绍了可观测性工具以及构建它们的作系统组件。

# 3.3 内核
以下各节讨论类 Unix 内核实现的详细信息，重点介绍性能。作为背景，讨论了早期内核的性能特性：Unix、BSD 和 Solaris。部分 3.4， Linux 中更详细地讨论了 Linux 内核。

内核的差异可能包括它们支持的文件系统（请参见第 8 章 “文件系统”）、系统调用 （syscall） 接口、网络堆栈体系结构、实时支持以及 CPU、磁盘 I/O 和网络的调度算法。

表 3.3 显示了 Linux 和其他内核版本以供比较，其中 syscall 计数基于作系统手册页第 2 节中的条目数。这是一个粗略的比较，但足以看到一些差异

这些只是带有文档的系统调用;更多的通常由内核提供，供作系统软件私人使用

UNIX 最初有 20 个系统调用，而今天 Linux（作为直接后代）已经有 1000 多个......我只是担心生长事物的复杂性和大小。Ken Thompson，ACM 图灵百年庆典，2012 年

Linux 的复杂性越来越高，通过添加新的 sys tem 调用或通过其他内核接口，将这种复杂性暴露给用户领域。额外的复杂性使学习、编程和调试更加耗时

# #3.3.1 unix
Unix 是由 Ken Thompson、Dennis Ritchie 和其他人在 AT&T 贝尔实验室于 1969 年及随后的几年中开发的。它的确切起源在 UNIX 分时系统 [Ritchie 74] 中进行了描述：

第一个版本是在我们中的一个人 （Thompson） 对可用的计算机设施不满意时编写的，他发现了一个很少使用的 PDP-7 并着手创造一个更友好的环境。

UNIX 的开发人员以前曾从事多路复用信息和计算机服务 （Multics）作系统的工作。UNIX 是作为轻量级多任务作系统和内核开发的，最初命名为 UNiplexed Information and Computing Service （UNICS），是 Multics 的双关语。来自 UNIX 实现 [Thompson 78]

内核是唯一不能被用户根据自己的喜好替换的 UNIX 代码。因此，内核应该尽可能少地做出实际决策。这并不意味着允许用户有一百万个选项来做同样的事情。相反，它的意思是只允许一种方法做一件事，但让该方式成为可能提供的所有选项的最小公约数。

虽然内核很小，但它确实提供了一些高性能功能。进程具有调度程序优先级，从而减少了更高优先级工作的运行队列延迟。为了提高效率，磁盘 I/O 以大（512 字节）块的形式出现，并缓存在内存中的每设备缓冲区缓存中。空闲的进程可以换出到存储中，从而允许更繁忙的进程在主内存中运行。当然，该系统是多任务处理的，允许多个进程同时运行，从而提高作业吞吐量。

为了支持网络、多文件系统、分页和我们现在认为的标准 stan dard 的其他功能，内核必须增长。随着多种衍生产品（包括 BSD、SunOS （Solaris） 和后来的 Linux）的出现，内核性能变得具有竞争力，这推动了更多功能和代码的添加。

# #3.3.2 BSD
Berkeley Software Distribution （BSD）作系统最初是加州大学伯克利分校 Unix 第 6 版的增强功能，于 1978 年首次发布。由于原始的 Unix 代码需要 AT&T 软件许可证，到 1990 年代初，这个 Unix 代码已经在新的 BSD 许可证下用 BSD 重写，允许包括 FreeBSD 在内的免费分发。

主要的 BSD 内核开发，尤其是与性能相关的开发，包括： 
■ 分页虚拟内存：BSD 为 Unix 带来了分页虚拟内存：可以移动（分页）较小的最近最少使用的内存块，而不是交换整个进程来释放主内存。请参见第 7 章 内存， 第 7.2.2 节 分页。
■ 按需分页：这将物理内存到虚拟内存的映射推迟到首次写入的时间，从而避免了可能永远不会使用的页面的早期和有时不必要的性能和内存成本。需求分页是由 BSD 引入 Unix 的。请参见第 7 章 内存， 第 7.2.2 节 分页。
■ FFS：Berkeley Fast File System （FFS） 将磁盘分配分组为柱面组，大大减少了碎片并提高旋转磁盘的性能，并支持更大的磁盘和其他增强功能。FFS 成为许多其他文件系统（包括 UFS）的基础。请参见第 8 章 “文件系统”， 部分 8.4.5， 文件系统类型。
■ TCP/IP 网络堆栈：BSD 为 Unix 开发了第一个高性能 TCP/IP 网络堆栈，包含在 4.2BSD（1983 年）中。BSD 仍然以其高性能的网络堆栈而闻名。
■ 套接字：Berkeley 套接字是连接端点的 API。它们包含在 4.2BSD 中，已成为网络标准。请参见第 10 章 “网络”。
■ Jails：轻量级作系统级虚拟化，允许多个 guest 共享一个内核。Jails 最初是在 FreeBSD 4.0 中发布的。
■ 内核 TLS：由于传输层安全性 （TLS） 现在在 Internet 上普遍使用，内核 TLS 将大部分 TLS 处理转移到内核，从而提高了性能14 [Stewart 15]

虽然 BSD 不如 Linux 流行，但它用于一些性能关键型环境，包括 Netflix 内容交付网络 （CDN） 以及来自 NetApp、Isilon 等的文件服务器。Netflix 将 2019 年 FreeBSD 在其 CDN 上的性能总结为 [Looney 19]：

“使用 FreeBSD 和商用部件，我们在 16 核 2.6 GHz CPU 上以 ~55% 的 CPU 实现了 90 Gb/s 的 TLS 加密连接服务。

有一本关于 FreeBSD 内部结构的优秀参考资料， 这本书的出版商是 The Design and Implementation of the FreeBSD Operating System， 2nd Edition [McKusick 15]。

# #3.3.3 solaris
Solaris 是 Sun Microsystems 于 1982 年创建的源自 Unix 和 BSD 的内核和作系统。它最初被命名为 SunOS，并针对 Sun 工作站进行了优化。到1980年代后期，AT&T基于SVR3、SunOS、BSD和Xenix的技术开发了新的Unix标准Unix System V Release 4 （SVR4）。Sun 基于 SVR4 创建了一个新内核，并以 Solaris 的名称重新命名了该作系统。

主要的 Solaris 内核开发，尤其是与性能相关的开发，包括： 
■ VFS：虚拟文件系统 （VFS） 是一种抽象和接口，允许多个文件系统轻松共存。Sun 最初创建它是为了让 NFS 和 UFS 可以共存。VFS 在第 8 章 “文件系统”中介绍。
■ 完全抢占式内核：这为高优先级工作（包括实时工作）提供了低延迟。
■ 多处理器支持：在 1990 年代初期，Sun 在多处理器作系统支持方面投入了大量资金，开发了对非对称和对称多处理（ASMP 和 SMP）的内核支持 [Mauro 01]。
■ Slab 分配器：内核 slab 内存分配器取代了 SVR4 伙伴分配器，通过每个 CPU 的预分配缓冲区缓存提供了更好的性能，这些缓存可以快速重用。这种分配器类型及其衍生物已成为包括 Linux 在内的 ker nels 的标准。
■ DTrace：一个静态和动态跟踪框架和工具，可在实时和生产环境中提供对整个软件堆栈的几乎无限的可观察性。Linux 具有用于此类可观察性的 BPF 和 bpftrace。
■ Zones：一种基于作系统的虚拟化技术，用于创建共享一个内核的作系统实例，类似于早期的 FreeBSD jails 技术。作系统虚拟化现在作为 Linux 容器得到广泛使用。请参见第 11 章 “云计算”。
■ ZFS：具有企业级功能和性能的文件系统。它现在可用于其他作系统，包括 Linux。请参见第 8 章 “文件系统”。

Oracle 于 2010 年收购了 Sun Microsystems，Solaris 现在称为 Oracle Solaris。本书的第一版中对 Solaris 进行了更详细的介绍

# 3.4 Linux
Linux 由 Linus Torvalds 于 1991 年创建，作为 Intel 个人计算机的免费作系统。他在 Usenet 的一篇博文中宣布了这个项目：

我正在为 386（486） AT 克隆制作一个（免费）作系统（只是一个爱好，不会像 gnu 那样大而专业）。这自 4 月以来一直在酝酿，并开始做好准备。我想听听大家对 minix 中喜欢/不喜欢的东西的任何反馈，因为我的作系统有点像它（文件系统的物理布局相同（由于实际原因）等）。

这指的是 MINIX作系统，它是作为用于小型计算机的 Unix 的免费小型（迷你）版本开发的。BSD 还旨在提供免费的 Unix 版本，尽管当时它遇到了法律问题

Linux 内核的开发采用了许多祖先的一般思想，包括： 
■ Unix（和 Multics）：作系统层、系统调用、多任务处理、进程、进程优先级、虚拟内存、全局文件系统、文件系统权限、设备节点、缓冲区缓存 
■ BSD：分页虚拟内存、需求分页、快速文件系统 （FFS）、TCP/IP 网络堆栈、套接字
■ Solaris： VFS、NFS、页面缓存、统一页面缓存、slab 分配器
■ 计划 9：资源分叉 （rfork），用于在进程和线程（任务）之间创建不同级别的共享

Linux 现在广泛用于服务器、云实例和嵌入式设备（包括移动电话）。

# #3.4.1 Linux内核发展
Linux 内核开发，尤其是与性能相关的内核开发，包括以下内容（其中许多描述包括首次引入它们的 Linux 内核版本）：CPU 调度类：已经开发了各种高级 CPU 调度算法，包括调度域 （2.6.7），以做出有关非一致性内存访问 （NUMA） 的更好决策。请参见第 6 章 “CPU”。
■ I/O 调度类：已经开发了不同的块 I/O 调度算法，包括截止时间 （2.5.39）、预期 （2.5.75） 和完全公平排队 （CFQ） （2.6.6）。这些在 Linux 5.0 之前的内核中可用，Linux 5.0 删除了它们以仅支持较新的多队列 I/O 调度器。请参见第 9 章 “磁盘”。
■ TCP 拥塞算法：Linux 允许配置不同的 TCP 拥塞控制算法，并在此列表中提到的后续内核中支持 Reno、Cubic 等。另请参见第 10 章 “网络”。
■ Over-Commit：除了内存不足 （OOM） 杀手之外，这是一种使用更少的主内存做更多事情的策略。参见第 7 章 “内存”。
■ Futex （2.5.7）：快速用户空间互斥锁的缩写，用于提供高性能的用户级同步原语。
■ 大页面 （2.5.36）：这为内核和内存管理单元 （MMU） 预分配的大内存页面提供支持。参见第 7 章 “内存”。■ OProfile （2.5.43）： 一个系统分析器，用于研究内核和应用程序的 CPU 使用率和其他事件。
■ RCU （2.5.43）：内核提供读取-副本更新同步机制，允许多次读取与更新同时进行，从而提高了大部分读取数据的性能和可扩展性。
■ epoll （2.5.46）： 一个系统调用，用于在许多打开的文件描述符中有效等待 I/O，这提高了服务器应用程序的性能。
■ BPF JIT （3.0）：Berkeley 数据包过滤器 （BPF） 的即时 （JIT） 编译器，通过将 BPF 字节码编译为本机指令来提高数据包过滤性能。
■ CFS 带宽控制 （3.2）：一种支持 CPU 配额和限流的 CPU 调度算法。
■ TCP 防缓冲膨胀 （3.3）：从 Linux 3.3 开始，为解决缓冲膨胀问题，进行了各种增强，包括用于数据包数据传输的字节队列限制 （BQL） （3.3）、CoDel 队列管理 （3.5）、TCP 小队列 （3.6） 和比例积分控制器增强型 （PIE） 数据包调度器 （3.14）。
■ uprobes （3.5）：用于动态跟踪用户级软件的基础设施，由其他工具（perf、SystemTap 等）使用。
■ TCP 早期重传 （3.5）：RFC 5827 用于减少触发快速重传所需的重复确认。
■ TFO（3.6、3.7、3.13）：TCP 快速打开 （TFO） 可以将 TCP 三次握手减少到带有 TFO Cookie 的单个 SYN 数据包，从而提高性能。它在 3.13 中成为默认值。
■ NUMA 平衡 （3.8）：这为内核增加了自动平衡多 NUMA 系统上内存位置的方法，从而减少 CPU 互连流量并提高性能。
■ SO_REUSEPORT （3.9）： 一个套接字选项，允许多个侦听器套接字绑定到同一个端口，从而提高多线程的可扩展性。
■ SSD 缓存设备 （3.9）：设备映射器支持将 SSD 设备用作旋转速度较慢的磁盘的缓存。
■ bcache （3.10）：用于块接口的 SSD 缓存技术。
■ TCP TLP （3.10）：TCP 尾部丢失探测 （TLP） 是一种方案，通过在较短的探测超时后发送新数据或最后一个未确认的段来避免昂贵的基于计时器的重新传输，以触发更快的恢复。
■ NO_HZ_FULL （3.10， 3.12）：也称为无计时器多任务或无滴答内核，这允许非空闲线程在没有时钟滴答声的情况下运行，从而避免工作负载扰动 [Corbet 13a]。
■ 多队列块 I/O （3.13）：这提供了每个 CPU 的 I/O 提交队列，而不是单个请求队列，从而提高了可扩展性，特别是对于高 IOPS SSD 设备 [Corbet 13b]。
■ SCHED_DEADLINE （3.14）： 一个可选的调度策略，用于实现最早截止时间优先 （EDF） 调度 [Linux 20b]。■ TCP 自动塞制 （3.14）：这允许内核合并小写入，从而减少发送的数据包。TCP_CORK 的自动版本 setsockopt（2）。
■ MCS 锁和 qspinlocks （3.15）：高效的内核锁，使用诸如 per CPU 结构等技术。MCS 以最初的锁发明者（Mellor-Crummey 和 Scott）[Mellor-Crummey 91][Corbet 14] 命名。
■ 扩展 BPF （3.18）：用于运行安全内核模式程序的内核内执行环境。大部分扩展 BPF 是在 4.x 系列中添加的。在 3.19 中添加了对 attached to kprobes 的支持，在 4.7 中添加了对跟踪点的支持，在 4.9 中添加了对软件和硬件事件的支持，在 4.10 中添加了对 cgroups 的支持。在 5.3 中添加了有界循环，这也增加了指令限制以允许复杂的应用程序。请参见部分 3.4.4， 扩展 BPF。
■ Overlayfs （3.18）：Linux 中包含的联合挂载文件系统。它在其他文件系统之上创建虚拟文件系统，也可以在不更改第一个文件系统的情况下对其进行修改。通常用于容器。
■ DCTCP （3.18）：数据中心 TCP （DCTCP） 拥塞控制算法，旨在提供高突发容忍度、低延迟和高吞吐量 [Borkmann 14a]。
■ DAX （4.0）：直接访问 （DAX） 允许用户空间直接从持久内存存储设备读取数据，而不会产生缓冲区开销。ext4 可以使用 DAX
■ 排队自旋锁 （4.2）：在争用时提供更好的性能，这些成为 4.2 中的默认自旋锁内核实现。
■ TCP 无锁侦听器 （4.4）：TCP 侦听器快速路径变为无锁，从而提高了性能。
■ cgroup v2 （4.5， 4.15）： cgroups 的统一层次结构在早期的内核中，在 4.5 中保持稳定并公开，命名为 cgroup v2 [Heo 15]。cgroup v2 CPU 控制器是在 4.15 中添加的。
■ epoll 可伸缩性 （4.5）：对于多线程可伸缩性，epoll（7） 避免了唤醒所有正在等待每个事件的相同文件描述符的线程，这会导致雷霆万钧的性能问题 [Corbet 15]。
■ KCM （4.6）：内核连接多路复用器 （KCM） 通过 TCP 提供基于消息的高效接口。
■ TCP NV （4.8）：New Vegas （NV） 是一种新的 TCP 拥塞控制算法，适用于高带宽网络（以 10 Gbps 运行的网络）。
■ XDP （4.8， 4.18）： eXpress 数据路径 （XDP） 是一种基于 BPF 的可编程快速路径，用于高性能网络 [Herbert 16]。在 4.18 中添加了一个可以绕过大部分网络堆栈的 AF_XDP 套接字地址系列。
■ TCP BBR （4.9）：瓶颈带宽和 RTT （BBR） 是一种 TCP 拥塞控制算法，可在遭受数据包丢失和缓冲区膨胀的网络上提供更好的延迟和吞吐量 [Cardwell 16]。
■ 硬件延迟跟踪器 （4.9）：一种 Ftrace 跟踪器，可以检测由硬件和固件引起的系统延迟，包括系统管理中断 （SMI）。
■ perf c2c （4.10）：缓存到缓存 （c2c） perf 子命令可以帮助识别 CPU 缓存性能问题，包括错误共享。
■ Intel CAT （4.10）：支持 Intel 高速缓存分配技术 （CAT），允许任务具有专用的 CPU 高速缓存空间。容器可以使用它来帮助解决 noisey neighbor 问题
■ 多队列 I/O 调度器：BPQ、Kyber （4.12）：Budget Fair Queueing （BFQ） 多队列 I/O 调度器为交互式应用程序提供低延迟 I/O，尤其是对于速度较慢的存储设备。BFQ 在 5.2 中得到了显著改善。Kyber I/O 调度器适用于快速多队列设备 [Corbet 17]。
■ 内核 TLS（4.13、4.17）：内核 TLS [Edge 15] 的 Linux 版本。
■ MSG_ZEROCOPY （4.14）： 一个 send（2） 标志，用于避免应用程序和网络接口之间额外的数据包字节副本 [Linux 20c]。
■ PCID （4.14）：Linux 增加了对进程上下文 ID （PCID） 的支持，这是一种处理器 MMU 功能，可帮助避免上下文切换时出现 TLB 刷新。这降低了缓解 meltdown 漏洞所需的内核页表隔离 （KPTI） 补丁的性能成本。请参见第 3.4.3 节 KPTI （Meltdown）。
■ PSI（4.20、5.2）：压力失速信息 （PSI） 是一组新指标，用于显示在 CPU、内存或 I/O 上花费的停顿时间。5.2 中添加了 PSI 阈值通知以支持 PSI 监控
■ TCP EDT （4.20）：TCP 堆栈切换到提前出发时间 （EDT）：它使用计时轮调度器来发送数据包，提供更好的 CPU 效率和更小的队列 [Jacobson 18]。
■ 多队列 I/O （5.0）：多队列块 I/O 调度器在 5.0 中成为默认值，并且删除了经典调度器。
■ UDP GRO （5.0）：UDP 通用接收卸载 （GRO） 通过允许数据包由驱动程序和卡聚合并向上传递堆栈来提高性能。
■ io_uring （5.1）： 一个通用的异步接口，用于应用程序与内核之间的快速通信，利用共享的环形缓冲区。主要用途包括快速磁盘和网络 I/O。
■ MADV_COLD、MADV_PAGEOUT （5.4）：这些 madvise（2） 标志是向内核提示需要内存，但不是很快需要。MADV_PAGEOUT也暗示了可以立即回收内存。这些对于内存受限的嵌入式 Linux 设备特别有用。
■ MultiPath TCP （5.6）：多个网络链接（例如 3G 和 WiFi）可用于提高单个 TCP 连接的性能和可靠性。
■ 引导时间跟踪 （5.6）：允许 Ftrace 跟踪早期引导过程。（systemd 可以提供有关延迟启动过程的计时信息：请参见 第 3.4.2 节 systemd。
■ 热压力 （5.7）：调度程序考虑了热限制，以做出更好的放置决策。
■ perf 火焰图 （5.8）：perf（1） 支持火焰图可视化。

这里没有列出许多小的性能改进，包括锁定、驱动程序、VFS、文件系统、异步 I/O、内存分配器、NUMA、新的处理器指令支持、GPU，以及性能工具 perf（1） 和 Ftrace。采用 systemd 也改善了系统启动时间

以下部分更详细地介绍了三个对性能很重要的 Linux 主题：systemd、KPTI 和扩展 BPF。

# #3.4.2 systemd
systemd 是 Linux 常用的服务管理器，是作为原始 UNIX init 系统的替代品而开发的。systemd 具有各种功能，包括依赖关系感知服务启动和服务时间统计。

系统性能中偶尔的任务是调整系统的引导时间，systemd 时间统计信息可以显示要调整的位置。总启动时间可以使用 systemd-analyze（1） 报告：

 # systemd-analyze
 Startup finished in 1.657s (kernel) + 10.272s (userspace) = 11.930s 
 graphical.target reached after 9.663s in userspac

此输出显示系统在 9.663 秒内启动（在本例中为 gital.target）。可以使用 critical-chain 子命令查看更多信息：

 # systemd-analyze critical-chain
 The time when unit became active or started is printed after the "@" character.
 The time the unit took to start is printed after the "+" character.
 graphical.target @9.663s
 └─multi-user.target @9.661s
    └─snapd.seeded.service @9.062s +62ms
        └─basic.target @6.336s
            └─sockets.target @6.334s
                └─snapd.socket @6.316s +16ms
                    └─sysinit.target @6.281s
                        └─cloud-init.service @5.361s +905ms
                            └─systemd-networkd-wait-online.service @3.498s +1.860s
                                └─systemd-networkd.service @3.254s +235ms
                                    └─network-pre.target @3.251s
                                        └─cloud-init-local.service @2.107s +1.141s
                                            └─systemd-remount-fs.service @391ms +81ms
                                                └─systemd-journald.socket @387ms
                                                    └─system.slice @366ms
                                                        └─-.slice @366ms

此输出显示关键路径：导致延迟的步骤序列（在本例中为 services）。最慢的服务是 systemd-networkd-wait-online.service，启动时间为 1.86 秒

还有其他有用的子命令： blame 显示最慢的初始化时间，plot 生成 SVG 图。有关更多信息，请参阅 systemd-analyze（1） 的手册页。

# #3.4.3 KPTI
2018 年添加到 Linux 4.14 的内核页表隔离 （KPTI） 补丁是为了缓解称为“meltdown”的 Intel 处理器漏洞。较旧的 Linux 内核版本具有用于类似目的的 KAISER 补丁，其他内核也采用了缓解措施。虽然这些方法可以解决安全问题，但由于上下文切换和系统调用上的额外 CPU 周期和额外的 TLB 刷新，它们也会降低处理器性能。Linux 在同一版本中添加了进程上下文 ID （PCID） 支持，只要处理器支持 pcid，就可以避免某些 TLB 刷新

我评估了 KPTI 对 Netflix 云生产工作负载的性能影响在 0.1% 到 6% 之间，具体取决于工作负载的系统调用速率（成本越高）[Gregg 18a]。额外的调整将进一步降低成本：使用大页面以便更快地预热 TLB，并使用跟踪工具检查系统调用以找出降低其速率的方法。许多此类跟踪工具是使用扩展的 BPF 实现的

# #3.4.4 扩展 BPF
BPF 代表 Berkeley Packet Filter，这是一种晦涩难懂的技术，于 1992 年首次开发，它提高了数据包捕获工具 [McCanne 92] 的性能。2013 年，Alexei Starovoitov 提出了对 BPF [Starovoitov 13] 的重大重写，该重写由他自己和 Daniel Borkmann 进一步开发，并于 2014 年包含在 Linux 内核中 [Borkmann 14b]。这使 BPF 变成了一个通用的执行引擎，可用于各种事情，包括网络工作、可观察性和安全性

BPF 本身是一种灵活高效的技术，由指令集、存储对象（映射）和辅助函数组成。由于其虚拟结构集规范，它可以被视为虚拟机。BPF 程序在内核模式下运行（如前面的图 3.2 所示），并配置为在事件上运行：套接字事件、跟踪点、USDT 探针、kprobe、uprobe 和 perf_events。这些如图 3.16 所示。

BPF 字节码必须首先通过检查安全性的验证器，确保 BPF 程序不会崩溃或损坏内核。它还可以使用 BPF 类型格式 （BTF） 系统来理解数据类型和结构。BPF 程序可以通过 perf ring 缓冲区（一种发出每个事件数据的有效方式）或通过 map（适用于统计）输出数据。

由于 BPF 为新一代高效、安全和先进的跟踪工具提供支持，因此对于系统性能分析非常重要。它为现有内核事件源提供可编程性：tracepoint、kprobes、uprobe 和 perf_events。例如，BPF 程序可以在 I/O 的开始和结束时记录时间戳，以计算其持续时间，并将其记录在自定义直方图中。本书包含许多使用 BCC 和 bpftrace 前端的基于 BPF 的程序。这些前端将在第 15 章中介绍。

# 3.5 其他建议
一些值得总结的其他内核和作系统主题是 PGO 内核、Unikernel、微内核、混合内核和分布式作系统。

# #3.5.1 PGO 内核
按配置优化 （PGO），也称为反馈导向优化 （FDO），使用 CPU 配置信息来改进编译器决策 [Yuan 14a]。这可以应用于内核构建，其中过程是：
1. 在生产环境中，获取 CPU 配置文件。
2. 根据该 CPU 配置文件重新编译内核。
3. 在生产环境中部署新内核。

这将为特定工作负载创建一个性能更高的内核。诸如 JVM 之类的运行时会自动执行此作，根据 Java 方法的运行时性能重新编译 Java 方法，并结合即时 （JIT） 编译。创建 PGO 内核的过程涉及手动步骤。

相关的编译优化是链接时优化 （LTO），其中一次编译整个二进制文件以允许在整个程序中进行优化。Microsoft Windows 内核大量使用 LTO 和 PGO，与 PGO [Bearman 20] 相比，性能提高了 5% 到 20%。Google 还使用 LTO 和 PGO 内核来提高性能 [Tolvanen 20]。

gcc 和 clang 编译器以及 Linux 内核都支持 PGO。内核 PGO 通常涉及运行专门检测的内核来收集配置文件数据。Google 发布了一个 AutoFDO 工具，它绕过了对这种特殊内核的需求：AutoFDO 允许使用 perf（1） 从普通内核中收集配置文件，然后将其转换为正确的格式供编译器使用 [Google 20a]。

最近唯一关于使用 PGO 或 AutoFDO 构建 Linux 内核的文档是 Microsoft [Bearman 20] 和 Google [Tolvanen 20] 在 2020 年 Linux 管道工大会上的两次演讲。

# #3.5.2 Unikernel
unikernel 是将内核、库和应用程序软件组合在一起的单一应用程序机器映像，通常可以在硬件 VM 或裸机上的单个地址空间中运行它。这可能具有性能和安全优势：更少的指令文本意味着更高的 CPU 缓存命中率和更少的安全漏洞。这也会产生一个问题：可能没有 SSH、shell 或性能工具可供您登录和调试系统，也没有任何方法可以添加它们。

为了使 unikernels 在生产环境中进行性能优化，必须构建新的性能工具和指标来支持它们。作为概念验证，我构建了一个基本的 CPU 分析器，它从 Xen dom0 运行，以分析 domU unikernel 客户机，然后构建一个 CPU 火焰图，只是为了证明它是可能的 [Gregg 16a]

unikernel 的示例包括 MirageOS [MirageOS 20]。

# #3.5.3 微内核
本章的大部分内容讨论了类 Unix 内核，也称为整体内核，其中管理设备的所有代码作为单个大型内核程序一起运行。对于微内核模型，内核软件保持在最低限度。微内核支持内存管理、线程管理和进程间通信 （IPC） 等基本功能。文件系统、网络堆栈和驱动程序作为用户模式软件实现，这使得这些用户模式组件可以更轻松地修改和替换。想象一下，不仅要选择要安装的数据库或 Web 服务器，还要选择要安装的网络堆栈。微内核还具有更强的容错能力：驱动程序崩溃不会使整个内核崩溃。微内核的示例包括 QNX 和 Minix 3

微内核的一个缺点是，执行 I/O 和其他功能需要额外的 IPC 步骤，从而降低性能。一种解决方案是混合内核，它结合了微内核和单片内核的优势。混合内核将性能关键型服务移回内核空间（使用直接函数调用而不是 IPC），就像使用单片内核一样，但保留了微内核的模块化设计和容错能力。混合内核的示例包括 Windows NT 内核和计划 9 内核。

# #3.5.4 分布式作系统
分布式作系统在一组联网的独立计算机节点上运行单个作系统实例。每个节点上通常使用一个微内核。分布式作系统的示例包括 Bell Labs 的 Plan 9 和 Inferno作系统。

虽然是一种创新设计，但该模型尚未得到广泛使用。Plan 9 和 Inferno 的共同创建者 Rob Pike 描述了造成这种情况的各种原因，包括 [Pike 00]

“在 1970 年代末和 1980 年代初，有人声称 Unix 扼杀了作系统研究，因为没有人会尝试其他任何事情。当时，我不相信。今天，我勉强接受这种说法可能是真的（尽管有 Microsoft）。

在云中，当今扩展计算节点的常见模型是在一组相同的作系统实例之间进行负载均衡，这些实例可以根据负载进行扩展（请参阅第 11 章 云计算， 第 11.1.3 节 容量规划）。

# 3.6 内核比较
哪个内核最快？这在一定程度上取决于 OS 配置和工作负载以及内核的参与程度。总的来说，我预计 Linux 的性能将优于其他内核，因为它在性能改进、应用程序和驱动程序支持、广泛使用以及发现和报告性能问题的大型社区方面做了广泛的工作。自 1993 年以来，TOP500 榜单跟踪的前 500 名超级公司在 2017 年变成了 100% 的 Linux [TOP500 17]。会有一些例外;例如，Netflix 使用云上的 Linux，将 FreeBSD 用于其 CDN。

通常使用微基准测试来比较内核性能，这很容易出错。此类基准测试可能会发现一个内核在特定 syscall 上要快得多，但该 syscall 未用于生产工作负载。（或者它使用，但带有某些未经过 microbenchmark 测试的标志，这会极大地影响性能。准确比较内核性能是高级性能工程师的任务，可能需要数周时间。请参阅第 12 章 基准测试， 第 12.3.2 节 主动基准测试，作为要遵循的方法。

在本书的第一版中，我在本节的结尾指出 Linux 没有成熟的动态跟踪器，没有它，您可能会错过巨大的性能优势。从第一版开始，我就转为全职 Linux 性能角色，并帮助开发了 Linux 所缺少的动态跟踪器：BCC 和 bpftrace，基于扩展的 BPF。这些在第15章和我的前一本书[格雷格书19章]中都有介绍。

第 3.4.1 节 Linux 内核开发列出了在第一版和此版本之间发生的许多其他 Linux 性能发展，涵盖内核版本 3.1 和 5.8。前面没有列出的一个主要发展是 OpenZFS 现在支持 Linux 作为其主内核，在 Linux 上提供高性能和成熟的文件系统选项。

然而，所有这些 Linux 开发也带来了复杂性。Linux 上有如此多的性能功能和可调参数，以至于为每个工作负载配置和调整它们变得很费力。我见过许多部署在未调整的情况下运行。在比较内核性能时，请记住这一点：每个内核都经过调优了吗？本书后面的章节及其调优部分可以帮助您解决这个问题

# 3.7 练习
1. 回答以下有关作系统术语的问题：
■ 进程、线程和任务有什么区别？
■ 什么是模式切换和上下文切换？
■ 分页和进程交换有什么区别？
■ I/O 密集型工作负载和 CPU 密集型工作负载有什么区别

2. 回答以下概念性问题： 
■ 描述 kernel 的作用。
■ 描述系统调用的角色。
■ 描述 VFS 的作用及其在 I/O 堆栈中的位置

3. 回答以下更深层次的问题： 
■ 列出线程离开当前 CPU 的原因。
■ 描述虚拟内存和需求分页的优势。

# 3.8 引用
 [Graham 68] Graham, B., “Protection in an Information Processing Utility,” Communications 
of the ACM, May 1968.
 [Ritchie 74] Ritchie, D. M., and Thompson, K., “The UNIX Time-Sharing System,” 
Communications of the ACM 17, no. 7, pp. 365–75, July 1974.
 [Thompson 78] Thompson, K., UNIX Implementation, Bell Laboratories, 1978. 
[Bach 86] Bach, M. J., The Design of the UNIX Operating System, Prentice Hall, 1986. 
 [Mellor-Crummey 91] Mellor-Crummey, J. M., and Scott, M., “Algorithms for Scalable 
Synchronization on Shared-Memory Multiprocessors,” ACM Transactions on Computing 
Systems, Vol. 9, No. 1, https://www.cs.rochester.edu/u/scott/papers/1991_TOCS_synch.pdf, 
1991.
 [McCanne 92] McCanne, S., and Jacobson, V., “The BSD Packet Filter: A New Architecture 
for User-Level Packet Capture”, USENIX Winter Conference, 1993.
 [Mills 94] Mills, D., “RFC 1589: A Kernel Model for Precision Timekeeping,” Network Working 
Group, 1994.
 [Lever 00] Lever, C., Eriksen, M. A., and Molloy, S. P., “An Analysis of the TUX Web Server,” 
CITI Technical Report 00-8, http://www.citi.umich.edu/techreports/reports/citi-tr-00-8.pdf, 
2000.
 [Pike 00] Pike, R., “Systems Software Research Is Irrelevant,” http://doc.cat-v.org/bell_labs/
 utah2000/utah2000.pdf, 2000.
 [Mauro 01] Mauro, J., and McDougall, R., Solaris Internals: Core Kernel Architecture, Prentice 
Hall, 2001.
 [Bovet 05] Bovet, D., and Cesati, M., Understanding the Linux Kernel, 3rd Edition, O’Reilly, 
2005.
 [Corbet 05] Corbet, J., Rubini, A., and Kroah-Hartman, G., Linux Device Drivers, 3rd Edition, 
O’Reilly, 2005.
 [Corbet 13a] Corbet, J., “Is the whole system idle?” LWN.net, https://lwn.net/Articles/
 558284, 2013.
 [Corbet 13b] Corbet, J., “The multiqueue block layer,” LWN.net, https://lwn.net/Articles/
 552904, 2013.
 [Starovoitov 13] Starovoitov, A., “[PATCH net-next] extended BPF,” Linux kernel mailing list, 
https://lkml.org/lkml/2013/9/30/627, 2013.
 [Borkmann 14a] Borkmann, D., “net: tcp: add DCTCP congestion control algorithm,” 
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/
 ?id=e3118e8359bb7c59555aca60c725106e6d78c5ce, 2014.
 [Borkmann 14b] Borkmann, D., “[PATCH net-next 1/9] net: filter: add jited flag to indicate 
jit compiled filters,” netdev mailing list, https://lore.kernel.org/netdev/1395404418-25376-1
git-send-email-dborkman@redhat.com/T, 2014.
 [Corbet 14] Corbet, J., “MCS locks and qspinlocks,” LWN.net, https://lwn.net/Articles/
 590243, 2014.
 [Drysdale 14] Drysdale, D., “Anatomy of a system call, part 2,” LWN.net, https://lwn.net/
 Articles/604515, 2014.
 [Yuan 14a] Yuan, P., Guo, Y., and Chen, X., “Experiences in Profile-Guided Operating 
System Kernel Optimization,” APSys, 2014.
 [Yuan 14b] Yuan P., Guo, Y., and Chen, X., “Profile-Guided Operating System Kernel 
Optimization,” http://coolypf.com, 2014.
 [Corbet 15] Corbet, J., “Epoll evolving,” LWN.net, https://lwn.net/Articles/633422, 2015.
 [Edge 15] Edge, J., “TLS in the kernel,” LWN.net, https://lwn.net/Articles/666509, 2015.
 [Heo 15] Heo, T., “Control Group v2,” Linux documentation, https://www.kernel.org/doc/
 Documentation/cgroup-v2.txt, 2015.
 [McKusick 15] McKusick, M. K., Neville-Neil, G. V., and Watson, R. N. M., The Design and 
Implementation of the FreeBSD Operating System, 2nd Edition, Addison-Wesley, 2015.
 [Stewart 15] Stewart, R., Gurney, J. M., and Long, S., “Optimizing TLS for High-Bandwidth 
Applicationsin FreeBSD,” AsiaBSDCon, https://people.freebsd.org/~rrs/asiabsd_2015_tls.pdf, 
2015.
 [Cardwell 16] Cardwell, N., Cheng, Y., Stephen Gunn, C., Hassas Yeganeh, S., and Jacobson, 
V., “BBR: Congestion-Based Congestion Control,” ACM queue, https://queue.acm.org/detail.
 cfm?id=3022184, 2016.
 [Gregg 16a] Gregg, B., “Unikernel Profiling: Flame Graphs from dom0,” http://
 www.brendangregg.com/blog/2016-01-27/unikernel-profiling-from-dom0.html, 2016.
 [Herbert 16] Herbert, T., and Starovoitov, A., “eXpress Data Path (XDP): Programmable and 
High Performance Networking Data Path,” https://github.com/iovisor/bpf-docs/raw/master/
 Express_Data_Path.pdf, 2016.
 [Corbet 17] Corbet, J., “Two new block I/O schedulers for 4.12,” LWN.net, https://lwn.net/
 Articles/720675, 2017.
 [TOP500 17] TOP500, “List Statistics,” https://www.top500.org/statistics/list, 2017.
 [Gregg 18a] Gregg, B., “KPTI/KAISER Meltdown Initial Performance Regressions,” http://
 www.brendangregg.com/blog/2018-02-09/kpti-kaiser-meltdown-performance.html, 2018.
 [Jacobson 18] Jacobson, V., “Evolving from AFAP: Teaching NICs about Time,” netdev 0x12, 
https://netdevconf.info/0x12/session.html?evolving-from-afap-teaching-nics-about-time, 
2018.
 [Gregg 19] Gregg, B., BPF Performance Tools: Linux System and Application Observability, 
Addison-Wesley, 2019.
 [Looney 19] Looney, J., “Netflix and FreeBSD: Using Open Source to Deliver Streaming 
Video,” FOSDEM, https://papers.freebsd.org/2019/fosdem/looney-netflix_and_freebsd, 2019.
 [Bearman 20] Bearman, I., “Exploring Profile Guided Optimization of the Linux Kernel,” 
Linux Plumber’s Conference, https://linuxplumbersconf.org/event/7/contributions/771, 2020.
 [Google 20a] Google, “AutoFDO,” https://github.com/google/autofdo, accessed 2020.
 [Linux 20a] “NO_HZ: Reducing Scheduling-Clock Ticks,” Linux documentation, https://
 www.kernel.org/doc/html/latest/timers/no_hz.html, accessed 2020.
 [Linux 20b] “Deadline Task Scheduling,” Linux documentation, https://www.kernel.org/doc/
 Documentation/scheduler/sched-deadline.rst, accessed 2020.
 [Linux 20c] “MSG_ZEROCOPY,” Linux documentation, https://www.kernel.org/doc/html/
 latest/networking/msg_zerocopy.html, accessed 2020.
 [Linux 20d] “Softlockup Detector and Hardlockup Detector (aka nmi_watchdog),” Linux 
documentation, https://www.kernel.org/doc/html/latest/admin-guide/lockup-watchdogs.
 html, accessed 2020.
 [MirageOS 20] MirageOS, “Mirage OS,” https://mirage.io, accessed 2020.
 [Owens 20] Owens, K., et al., “4. Kernel Stacks,” Linux documentation, https://www.kernel.org/
 doc/html/latest/x86/kernel-stacks.html, accessed 2020.
 [Tolvanen 20] Tolvanen, S., Wendling, B., and Desaulniers, N., “LTO, PGO, and AutoFDO 
in the Kernel,” Linux Plumber’s Conference, https://linuxplumbersconf.org/event/7/ 
contributions/798, 2020.

# #3.8.1 额外阅读
操作系统及其内核是一个引人入胜且广泛的话题。本章仅总结了基本内容。除了本章中提到的来源外，以下是适用于基于 Linux 的作系统和其他作系统的优秀参考资料：
 [Goodheart 94] Goodheart, B., and Cox J., The Magic Garden Explained: The Internals of UNIX 
System V Release 4, an Open Systems Design, Prentice Hall, 1994. 
 [Vahalia 96] Vahalia, U., UNIX Internals: The New Frontiers, Prentice Hall, 1996.
 [Singh 06] Singh, A., Mac OS X Internals: A Systems Approach, Addison-Wesley, 2006.
  [McDougall 06b] McDougall, R., and Mauro, J., Solaris Internals: Solaris 10 and OpenSolaris 
Kernel Architecture, Prentice Hall, 2006.
 [Love 10] Love, R., Linux Kernel Development, 3rd Edition, Addison-Wesley, 2010.
 [Tanenbaum 14] Tanenbaum, A., and Bos, H., Modern Operating Systems, 4th Edition, 
Pearson, 2014.
 [Yosifovich 17] Yosifovich, P., Ionescu, A., Russinovich, M. E., and Solomon, D. A., Windows 
Internals, Part 1 (Developer Reference), 7th Edition, Microsoft Press, 2017.

