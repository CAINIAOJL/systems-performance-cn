# Chapter 4: 可观测工具

操作系统历来提供了许多用于观察系统软件和硬件组件的工具。对于新手来说，广泛的可用工具和指标表明，所有东西（或者至少是所有重要的东西）都可以被观察到。实际上存在许多差距，系统性能专家精通推理和交互艺术：从间接工具和统计数据中找出活动。例如，可以单独检查网络数据包 （嗅探），但磁盘 I/O 不能 （至少不容易）。

由于动态跟踪工具的兴起，Linux 中的可观测性得到了极大的提高，包括基于 BPF 的 BCC 和 bpftrace。暗角现在被照亮，包括使用 biosnoop（8） 的单个磁盘 I/O。然而，许多公司和商业监控产品尚未采用系统跟踪，并且错过了它带来的洞察力。我通过开发、发布和解释新的追踪工具引领潮流，这些工具已经被 Netflix 和 Facebook 等公司使用

本章的学习目标是： 
■ 识别静态性能工具和危机工具。
■ 了解工具类型及其开销：计数器、分析和跟踪。
■ 了解可观测性来源，包括：/proc、/sys、tracepoints、kprobes、uprobes、USDT 和 PMC。
■ 了解如何配置 sar（1） 以存档统计数据。

在第 1 章中，我介绍了不同类型的可观测性：计数器、分析和跟踪，以及静态和动态插桩。本章详细介绍了可观测性工具及其数据源，包括 sar（1） 摘要、系统活动报告程序和跟踪工具简介。这为您提供了了解 Linux 可观察性的基础知识;后面的章节（6 到 11）使用这些工具和资源来解决特定问题。第 13 章到第 15 章深入介绍了示踪剂。

本章以 Ubuntu Linux 发行版为例;这些工具中的大多数在其他 Linux 发行版中是相同的，并且这些工具起源的其他内核和作系统也存在一些类似的工具。

# 4.1 工具预览
图 4.1 显示了一个作系统图，我使用与每个组件相关的 Linux 工作负载可观察性工具1 对其进行了注释。

这些工具中的大多数都专注于特定资源，例如 CPU、内存或磁盘，并在后面专门介绍该资源的章节中介绍。有一些多工具可以分析许多区域，本章稍后将介绍它们：perf、Ftrace、BCC 和 bpftrace。

# #4.1.1 静态性能工具
还有另一种类型的可观测性，它检查系统在静态而不是活动工作负载下的属性。这在第 2 章 方法论， 第 2.5.17 节 静态性能调优中被描述为静态性能调优方法，这些工具如图 4.2 所示。

请记住使用图 4.2 中的工具来检查配置和组件的问题。有时，性能问题仅仅是由于配置错误造成的。

# #4.1.2 危机工具
当您遇到需要各种性能工具进行调试的生产性能危机时，您可能会发现没有安装任何工具。更糟糕的是，由于服务器遇到性能问题，安装工具可能需要比平时更长的时间，从而延长危机。
对于 Linux，表 4.1 列出了推荐用于这些危机工具的安装包或源存储库。此表显示了 Ubuntu/Debian 的软件包名称（这些软件包名称可能因不同的 Linux 发行版而异）。

大公司（如 Netflix）拥有作系统和性能团队，他们确保生产系统安装了所有这些软件包。默认的 Linux 发行版可能只安装了 procps 和 util-linux，因此必须添加所有其他发行版。

在容器环境中，可能需要创建一个特权调试容器，该容器对 system2 和安装的所有工具具有完全访问权限。此容器的映像可以安装在容器主机上，并在需要时进行部署

添加工具包通常是不够的：内核和用户空间软件可能还需要配置为支持这些工具。跟踪工具通常需要启用某些内核 CONFIG 选项，例如 CONFIG_FTRACE 和 CONFIG_BPF。分析工具通常需要将软件配置为支持堆栈遍历，方法是使用所有软件（包括系统库：libc、libpthread 等）的帧指针编译版本或安装的 debuginfo 软件包以支持 Dwarf Stack Walking。如果您的公司还没有这样做，您应该检查每个性能工具是否有效，并在危机中迫切需要之前修复那些无效的工具

以下部分更详细地介绍了性能可观测性工具

# 4.2 工具类型
可观测性工具的一个有用分类是它们是否提供系统范围的或每个进程的可观测性，以及它们是基于计数器还是基于事件。这些属性如图 4.3 所示，并附有 Linux 工具示例。

有些工具适合多个象限;例如，top（1） 也有一个系统范围的摘要，系统范围的事件工具通常可以过滤一个特定的进程 （-p PID）。

基于事件的工具包括 Profiler 和 Tracer。分析器通过对事件拍摄一系列快照来观察活动，从而绘制目标的粗略图片。Tracers 检测每个感兴趣的事件，并可以对它们执行处理，例如生成自定义计数器。计数器、跟踪和分析在第 1 章中介绍

以下部分介绍使用固定计数器、跟踪和分析的 Linux 工具，以及执行监控 （指标） 的 Linux 工具。

# #4.2.1 固定计数器
内核维护各种计数器以提供系统统计信息。它们通常实现为无符号整数，当事件发生时，这些整数会递增。例如，存在用于接收的网络数据包数、发出的磁盘 I/O 数和发生的中断的计数器。这些指标由监控软件作为度量标准公开（请参见部分 4.2.4， 监控）。

一种常见的内核方法是维护一对累积计数器：一个用于对事件进行计数，另一个用于记录事件中的总时间。这些直接提供事件计数以及事件中的平均时间（或延迟），方法是将总时间除以计数。由于它们是累积的，因此通过以一定的时间间隔（例如，1 秒）读取对，可以计算出 delta，并从中计算出每秒计数和平均延迟。这是计算系统统计信息的数量

在性能方面，计数器被认为是 “免费” 使用的，因为它们默认启用并由内核持续维护。使用它们时唯一的额外成本是从用户空间读取它们的值的行为（这应该可以忽略不计）。以下示例工具在系统范围内或按进程读取这些内容

# 系统范围
这些工具使用内核计数器在系统软件或硬件资源的上下文中检查系统范围的活动。Linux 工具包括： 
■ vmstat（8）：虚拟和物理内存统计，系统范围 
■ mpstat（1）：每个 CPU 使用率 
■ iostat（1）：每个磁盘的 I/O 使用率，从块设备接口 
■ nstat（8） 报告：TCP/IP 堆栈统计 
■ sar（1）：各种统计数据;还可以存档它们以进行历史报告

这些工具通常可供系统上的所有用户（非 root）查看。它们的统计数据通常也由监控软件绘制成图表。

许多应用程序都遵循一个使用约定，它们接受可选的 interval 和 count，例如，vmstat（8） 的 interval 为 1 秒，output count 为 3：

 # $ vmstat 1 3
 procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 4  0 1446428 662012 142100 5644676    1    4    28   152   33    1 29  8 63  0  0
 4  0 1446428 665988 142116 5642272    0    0     0   284 4957 4969 51  0 48  0  0
 4  0 1446428 685116 142116 5623676    0    0     0     0 4488 5507 52  0 48  0  0

输出的第一行是 summary-since-boot，它显示系统启动的整个时间的平均值。后续行是一秒间隔摘要，显示当前活动。至少，这是它的意图：这个 Linux 版本混合了第一行的 summary-since-boot 和 current 值（memory 列是 current 值;vmstat（8） 在第 7 章中有解释）。

# 每进程
这些工具是面向进程的，并使用内核为每个进程维护的计数器。Linux 工具包括：
■ ps（1）：显示进程状态，显示各种进程统计信息，包括内存和 CPU 使用率。
■ top（1）：显示排名靠前的进程，按 CPU 使用率或其他统计数据排序。
■ pmap（1）：列出带有使用情况统计信息的进程内存段。
这些工具通常从 /proc 文件系统读取统计信息

# #4.2.2 分析
性能分析通过收集目标行为的一组样本或快照来描述目标的特征。CPU 使用率是性能分析的常见目标，其中对指令指针或堆栈跟踪进行基于计时器的采样，以描述消耗 CPU 的代码路径。这些样本通常以固定速率收集，例如在所有 CPU 上以 100 Hz（每秒周期数）收集，并持续较短的持续时间，例如一分钟。分析工具（或分析器）通常使用 99 Hz 而不是 100 Hz，以避免与目标活动同步采样，这可能会导致计数过高或计数不足

性能分析还可以基于未定时的硬件事件，例如 CPU 硬件高速缓存未命中或总线活动。它可以显示哪些代码路径是负责的，这些信息特别可以帮助开发人员优化其代码的内存使用情况。

与固定计数器不同，性能分析（和跟踪）通常仅根据需要启用，因为它们可能会花费一些 CPU 开销来收集，并花费一些存储开销。这些开销的大小取决于工具及其检测的事件速率。基于计时器的配置文件通常更安全：事件速率是已知的，因此可以预测其开销，并且可以选择事件速率以具有可忽略不计的开销。

# 系统范围
系统范围的 Linux 分析器包括： 
■ perf（1）：标准的 Linux 分析器，包括 profiling 子命令。
■ profile（8）：来自 BCC 存储库的基于 BPF 的 CPU 分析器（在第 15 章 BPF 中介绍），用于对内核上下文中的堆栈跟踪进行频率计数。
■ 英特尔 VTune Amplifier XE：Linux 和 Windows 分析，具有包括源代码浏览在内的图形界面。

这些也可以用于定位单个进程。

# 每进程
面向进程的分析器包括：
■ gprof（1）：GNU 分析工具，用于分析由编译器（例如 gcc -pg）添加的分析信息。
■ cachegrind：valgrind 工具包中的一个工具，可以使用 kcachegrind 分析硬件缓存使用情况（以及更多）并可视化配置文件。
■ Java Flight Recorder （JFR）：编程语言通常有自己的专用分析器，可以检查语言上下文。例如，用于 Java 的 JFR。

有关分析工具的更多信息，请参见第 6 章 “CPU”和第 13 章 “perf”。

# #4.2.3 追踪
跟踪会检测事件的每次发生，并且可以存储基于事件的详细信息以供以后分析或生成摘要。这类似于分析，但目的是收集或检查所有事件，而不仅仅是样本。与分析相比，跟踪可能会产生更高的 CPU 和存储开销，这可能会减慢跟踪目标的速度。应该考虑到这一点，因为它可能会对生产工作负载产生负面影响，并且测得的时间戳也可能被跟踪器扭曲。与性能分析一样，跟踪通常仅在需要时使用。

日志记录（其中错误和警告等不常见的事件）被写入日志文件以供以后阅读，可以被视为默认启用的低频跟踪。日志包括系统日志。

以下是系统范围和每个进程的跟踪工具的示例

# 系统范围
这些跟踪工具使用内核跟踪工具在系统软件或硬件资源的上下文中检查系统范围的活动。Linux 工具包括： 
■ tcpdump（8）：网络数据包跟踪（使用 libpcap） 
■ biosnoop（8）：块 I/O 跟踪（使用 BCC 或 bpftrace） 
■ execsnoop（8）：新进程跟踪（使用 BCC 或 bpftrace） 
■ perf（1）：标准 Linux 分析器，也可以跟踪事件 
■ perf trace：一个特殊的 perf 子命令，用于跟踪系统范围的 
■ Ftrace系统调用： Linux 内置跟踪器 BCC：BPF 的跟踪库和工具包 
■ bpftrace：基于 BPF 的跟踪器 （bpftrace（8）） 和工具包

perf（1）、Ftrace、BCC 和 bpftrace 在第 4.5 节 跟踪工具中介绍，并在第 13 章到第 15 章中详细介绍。使用 BCC 和 bpftrace 构建的跟踪工具有 100 多个，包括 biosnoop（8） 和 execsnoop（8）。本书中提供了更多示例。

# 每进程
这些跟踪工具是面向进程的，它们所基于的作系统框架也是如此。Linux 工具包括：
■ strace（1）： 系统调用跟踪 
■ gdb（1）： 源代码级调试器

调试器可以检查每个事件的数据，但它们必须通过停止和启动目标的执行来实现。这可能会带来巨大的间接成本，使其不适合生产使用。

系统范围的跟踪工具（如 perf（1） 和 bpftrace）支持用于检查单个进程的过滤器，并且可以以低得多的开销运行，因此在可用的情况下使其成为首选工具。

# #4.2.4 监控
监视在第 2 章 方法论中介绍。与前面介绍的工具类型不同，监控会持续记录统计数据，以备日后需要时使用

# sar(1)
用于监控单个作系统主机的传统工具是系统活动报告器，sar（1），它源自 AT&T Unix。sar（1） 是基于计数器的，并且有一个代理程序，该代理程序在计划的时间（通过 cron）执行，以记录系统范围的计数器的状态。sar（1） 工具允许在命令行中查看这些内容，例如：

 # sar
 Linux 4.15.0-66-generic (bgregg)  12/21/2019        _x86_64_        (8 CPU)
 12:00:01 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
 12:05:01 AM     all      3.34      0.00      0.95      0.04      0.00     95.66
 12:10:01 AM     all      2.93      0.00      0.87      0.04      0.00     96.16
 12:15:01 AM     all      3.05      0.00      1.38      0.18      0.00     95.40
 12:20:01 AM     all      3.02      0.00      0.88      0.03      0.00     96.06
 [...]
 Average:        all      0.00      0.00      0.00      0.00      0.00      0.0

默认情况下，sar（1） 读取其 statistics 存档（如果启用）以打印最近的历史统计信息。您可以指定可选的间隔和计数，以便它以指定的速率检查当前活动

sar（1） 可以记录数十种不同的统计数据，以深入了解 CPU、内存、磁盘、网络工作、中断、电源使用情况等。第 4.4 节 sar 中对此进行了更详细的介绍。

第三方监控产品通常基于 sar（1） 或它使用的相同可观测性统计数据构建，并通过网络公开这些指标。

# SNMP
网络监控的传统技术是简单网络管理协议 （SNMP）。设备和作系统可以支持 SNMP，并且在某些情况下默认提供 SNMP，无需安装第三方代理或导出器。SNMP 包括许多基本的作系统指标，尽管它尚未扩展到涵盖现代应用程序。大多数环境已改用基于自定义代理的监控。

# Agents
现代监控软件在每个系统上运行代理（也称为导出器或插件）来记录内核和应用程序指标。这些可以包括特定应用程序和目标的代理，例如 MySQL 数据库服务器、Apache Web 服务器和 Memcached 缓存系统。此类代理可以提供单独的系统计数器无法提供的详细应用程序请求指标

适用于 Linux 的监控软件和代理包括：
■ 性能副驾驶 （PCP）：PCP 支持数十种不同的代理（称为性能指标域代理：PMDA），包括基于 BPF 的指标 [PCP 20]。■ ■ Prometheus：Prometheus 监控软件支持数十种不同的导出器，用于数据库、硬件、消息传递、存储、HTTP、API 和日志记录 [Prometheus 20]。
■ collectd：支持数十种不同的插件

图 4.4 中显示了一个示例监控架构，其中涉及一个用于存档指标的监控数据库服务器和一个用于提供客户端 UI 的监控 Web 服务器。这些指标由代理发送（或提供）到数据库服务器，然后提供给客户端 UI，以折线图和仪表板的形式显示。例如，Graphite Carbon 是一个监控数据库服务器，而 Grafana 是一个监控 Web 服务器/仪表板。

有数十种监控产品，以及针对不同目标类型的数百种不同代理。涵盖它们超出了本书的范围。但是，这里涵盖了一个共同的 denom inator：系统统计信息（基于内核计数器）。监控产品显示的系统统计信息通常与系统工具（vmstat（8）、iostat（1） 等）显示的系统统计信息相同。了解这些将帮助您了解监控产品，即使您从未使用过命令行工具。这些工具将在后面的章节中介绍

一些监控产品通过运行系统工具并解析文本输出来读取其系统指标，这种效率很低。Better Monitoring 产品使用库和内核接口直接读取指标 — 与命令行工具使用的界面相同。下一节将介绍这些来源，重点介绍最常见的因素：内核接口。

# 4.3 观测资源
接下来介绍系统性能统计信息的主要来源：/proc 和 /sys。然后涵盖了其他 Linux 源：延迟记帐、netlink、tracepoints、kprobes、USDT、uprobes、PMC 等。

第 13 章 perf、第 14 章 Ftrace 和第 15 章 BPF 中介绍的跟踪器利用了其中的许多源，尤其是系统范围的跟踪。这些跟踪源的范围如图 4.5 所示，还有事件和组名称：例如，block： 用于所有块 I/O 跟踪点，包括 block：block_rq_issue

图 4.5 中仅显示了几个示例 USDT 来源，用于 PostgreSQL 数据库 （ postgres：）、JVM 热点编译器 （hotspot：） 和 libc （libc：）。根据您的用户级软件，您可能还有更多。

有关 tracepoint、kprobe 和 uprobe 如何工作的更多信息，它们的内部内容记录在 BPF 性能工具 [Gregg 19] 的第 2 章中。

# #4.3.1 /proc
这是内核统计信息的文件系统接口。/proc 包含许多目录，其中每个目录都以其所代表的进程的进程 ID 命名。在每一个指令中，都有许多文件，其中包含从内核数据结构映射的每个进程的信息和统计数据。/proc 中还有其他文件用于系统范围的统计信息。

/proc 由内核动态创建，不受存储设备支持（它在内存中运行）。它大部分是只读的，为可观测性工具提供统计信息。某些文件是可写的，用于控制进程和内核行为

文件系统接口很方便：它是一个直观的框架，用于通过目录树向用户公开内核统计信息，并且通过 POSIX 文件系统调用具有众所周知的编程接口：open（）、read（）、close（）。你也可以在命令行中使用 cd、cat（1）、grep（1） 和 awk（1） 来探索它。文件系统还通过使用文件访问权限提供用户级安全性。在极少数情况下，典型的进程可观察性工具（ps（1）、top（1） 等）无法执行，某些进程调试仍然可以由 /proc 目录中的 shell 内置工具执行。

读取大多数 /proc 文件的开销可以忽略不计;例外情况包括一些遍历页表的内存映射相关文件。

# 每进程统计信息
/proc 中提供了各种文件，用于每个进程的统计信息。下面是一个可能可用的示例 （Linux 5.4），这里查看 PID 18733

 $ ls -F /proc/18733
 arch_status      environ     mountinfo      personality   statm
 attr/            exe@        mounts         projid_map    status
 autogroup        fd/         mountstats     root@         syscall
 auxv             fdinfo/     net/           sched         task/
 cgroup           gid_map     ns/            schedstat     timers
 clear_refs       io          numa_maps      sessionid     timerslack_ns
 cmdline          limits      oom_adj        setgroups     uid_map
 comm             loginuid    oom_score      smaps         wchan
 coredump_filter  map_files/  oom_score_adj  smaps_rollup
 cpuset           maps        pagemap        stack
 cwd@             mem         patch_state    stat

可用文件的确切列表取决于内核版本和 CONFIG 选项。与每个进程的性能可观察性相关的选项包括：

■ limits： 有效资源限制 
■ maps：映射的内存区域 
■ sched ：各种 CPU 调度程序统计信息 
■ schedstat：CPU 运行时间、延迟和时间片 
■ smap： 具有使用情况统计的映射内存区域 
■ stat： 进程状态和统计信息，包括总 CPU 和内存使用情况 
■ statm： 以页面为单位的内存使用情况摘要 
■ status： stat 和 statm 信息，标记为 
■ fd：文件描述符符号链接的目录（另请参阅 fdinfo）
■ cgroup：Cgroup 成员资格信息 
■ task：每个任务（线程）统计信息的目录

下面显示了 top（1） 如何读取每个进程的统计信息，并使用 strace（1） 进行跟踪：

 stat("/proc/14704", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
 open("/proc/14704/stat", O_RDONLY)      = 4
 read(4, "14704 (sshd) S 1 14704 14704 0 -"..., 1023) = 232
 close(4)  

这将在以进程 ID （14704） 命名的目录中打开一个名为“stat”的文件，然后读取文件内容。

top（1） 对系统上的所有活动进程重复此作。在某些系统上，尤其是那些具有许多进程的系统上，执行这些作的开销可能会变得很明显，特别是对于 top（1） 版本，它们在每次屏幕更新时对每个进程重复此序列。这可能会导致 top（1） 报告 top 本身是最高的 CPU 使用者的情况！

# 系统范围的统计信息
Linux 还扩展了 /proc 以包含系统范围的统计信息，包含在以下附加文件和目录中：

 # $ cd /proc; ls -Fd [a-z]*
 acpi/      dma          kallsyms     mdstat        schedstat      thread-self@
 buddyinfo  driver/      kcore        meminfo       scsi/          timer_list
 bus/       execdomains  keys         misc          self@          tty/
 cgroups    fb           key-users    modules       slabinfo       uptime
 cmdline    filesystems  kmsg         mounts@       softirqs       version
 consoles   fs/          kpagecgroup  mtrr          stat           vmallocinfo
 cpuinfo    interrupts   kpagecount   net@          swaps          vmstat
 crypto     iomem        kpageflags   pagetypeinfo  sys/           zoneinfo
 devices    ioports      loadavg      partitions    sysrq-trigger
 diskstats  irq/         locks        sched_debug   sysvipc/

与性能可观测性相关的系统范围文件包括

■ cpuinfo：物理处理器信息，包括每个虚拟 CPU、型号名称、时钟速度和缓存大小。
■ diskstats： 所有磁盘设备的磁盘 I/O 统计信息 
■ 中断： 每个 CPU 负载的中断计数器
■ avg： 负载平均值 
■ meminfo： 系统内存使用明细 
■ net/dev： 网络接口统计信息 
■ net/netstat： 系统范围的网络统计信息 
■ net/tcp： 活动 TCP 套接字信息 
■ pressure/： 压力停顿信息 （PSI） 文件 
■ schedstat： 系统范围的 CPU 调度程序统计信息 
■ self： 指向当前进程 ID 目录的符号链接， 为方便起见，
■ slabinfo： 内核 slab 分配器缓存统计信息 
■ stat： 内核和系统资源统计信息的摘要： CPU、磁盘、分页、交换、进程 
■ zoneinfo： 内存区域信息

这些由系统范围的工具读取。例如，下面是 vmstat（8） 读取 /proc，由 strace（1） 跟踪：

 open("/proc/meminfo", O_RDONLY)         = 3
 lseek(3, 0, SEEK_SET)                   = 0
 read(3, "MemTotal:         889484 kB\nMemF"..., 2047) = 1170
 open("/proc/stat", O_RDONLY)            = 4
 read(4, "cpu  14901 0 18094 102149804 131"..., 65535) = 804
 open("/proc/vmstat", O_RDONLY)          = 5
 lseek(5, 0, SEEK_SET)                   = 0
 read(5, "nr_free_pages 160568\nnr_inactive"..., 2047) = 1998

此输出显示 vmstat（8） 正在读取 meminfo、stat 和 vmstat。

# CPU 统计准确性
/proc/stat 文件提供系统范围的 CPU 利用率统计信息，并被许多工具（vmstat（8）、mpstat（1）、sar（1）、监视代理）使用。这些统计信息的准确性取决于内核配置。默认配置 （CONFIG_TICK_CPU_ACCOUNTING）使用时钟周期的粒度 [Weisbecker 13] 来测量 CPU 利用率，可能是 4 毫秒（取决于 CONFIG_HZ）。这通常就足够了。有一些选项可以通过使用更高分辨率的计数器来提高准确性，尽管性能成本较低（VIRT_CPU_ACCOUNTING_NATIVE 和 VIRT_CPU_ACCOUTING_GEN），也可以选择更准确的 IRQ 时间 （IRQ_TIME_ACCOUNTING）。获得准确 CPU 利用率测量值的另一种方法是使用 MSR 或 PMC

# 文件内容
/proc 文件通常采用文本格式，以便从命令行轻松读取并由 shell 脚本工具进行处理。例如：

# $ cat /proc/meminfo 
MemTotal:       15923672 kB
 MemFree:        10919912 kB
 MemAvailable:   15407564 kB
 Buffers:           94536 kB
 Cached:          2512040 kB
 SwapCached:            0 kB
 Active:          1671088 kB
 [...]
# $ grep Mem /proc/meminfo 
MemTotal:       15923672 kB
 MemFree:        10918292 kB
 MemAvailable:   15405968 kB

虽然这很方便，但它确实为内核将统计数据编码为文本以及随后解析文本的任何用户空间工具增加了少量开销。netlink 在第 4.3.4 节 netlink 中介绍，是一个更高效的二进制接口。

/proc 的内容记录在 proc（5） 手册页和 Linux 内核文档中：Documentation/filesystems/proc.txt [Bowden 20]。部分有扩展的文档，比如 Documentation/iostats.txt 中的 diskstats 和 Documentation/ scheduler/sched-stats.txt 中的 scheduler stats。除了文档之外，您还可以研究内核源代码以了解 /proc 中所有项目的确切来源。将源代码读取到使用它们的工具也很有帮助。

某些 /proc 条目取决于 CONFIG 选项：schedstats 使用 CONFIG_ SCHEDSTATS 启用，sched 与 CONFIG_SCHED_DEBUG 启用，pressure 与 CONFIG_PSI 启用。

# #4.3.2 /sys
Linux 提供了一个 sysfs 文件系统，挂载在 /sys 上，它是在 2.6 内核中引入的，用于为内核统计信息提供基于目录的结构。这与 /proc 不同，后者随着时间的推移而发展，并且大多数将各种系统统计信息添加到顶级目录中。sysfs 最初旨在提供设备驱动程序统计信息，但已扩展为包括任何统计信息类型。

例如，下面列出了 CPU 0（截断）的 /sys 文件：
 # $ find /sys/devices/system/cpu/cpu0 -type f
 /sys/devices/system/cpu/cpu0/uevent
 /sys/devices/system/cpu/cpu0/hotplug/target
 /sys/devices/system/cpu/cpu0/hotplug/state
 /sys/devices/system/cpu/cpu0/hotplug/fail
 /sys/devices/system/cpu/cpu0/crash_notes_size
 /sys/devices/system/cpu/cpu0/power/runtime_active_time
 /sys/devices/system/cpu/cpu0/power/runtime_active_kids
 /sys/devices/system/cpu/cpu0/power/pm_qos_resume_latency_us
 /sys/devices/system/cpu/cpu0/power/runtime_usage
 [...]
 /sys/devices/system/cpu/cpu0/topology/die_id
 /sys/devices/system/cpu/cpu0/topology/physical_package_id
 /sys/devices/system/cpu/cpu0/topology/core_cpus_list
 /sys/devices/system/cpu/cpu0/topology/die_cpus_list
 /sys/devices/system/cpu/cpu0/topology/core_siblings
 [...]

列出的许多文件都提供有关 CPU 硬件缓存的信息。下面的输出显示了它们的内容（使用 grep（1） 以便文件名包含在输出中）：

# $ grep . /sys/devices/system/cpu/cpu0/cache/index*/level 
/sys/devices/system/cpu/cpu0/cache/index0/level:1
 /sys/devices/system/cpu/cpu0/cache/index1/level:1
 /sys/devices/system/cpu/cpu0/cache/index2/level:2
 /sys/devices/system/cpu/cpu0/cache/index3/level:3
# $ grep . /sys/devices/system/cpu/cpu0/cache/index*/size  
/sys/devices/system/cpu/cpu0/cache/index0/size:32K
 /sys/devices/system/cpu/cpu0/cache/index1/size:32K
 /sys/devices/system/cpu/cpu0/cache/index2/size:1024K
 /sys/devices/system/cpu/cpu0/cache/index3/size:33792K

这表明 CPU 0 可以访问两个 1 级缓存，每个缓存 32 KB，一个 2 级缓存为 1 MB，一个 3 级缓存为 33 MB。

/sys 文件系统通常在只读文件中包含数万个统计信息，以及许多用于更改内核状态的可写文件。例如，可以通过将 “1” 或 “0” 写入名为 “online” 的文件，将 CPU 设置为在线或离线。与读取统计信息一样，可以通过在命令行中使用文本字符串 （echo 1 > filename） 而不是二进制接口来进行某些状态设置。

# #4.3.3 延迟核算
具有 CONFIG_TASK_DELAY_ACCT 选项的 Linux 系统在以下状态下跟踪每个任务的时间：

■ 调度程序延迟：等待开机 CPU 
■ 块 I/O：等待块 I/O 完成 
■ 交换：等待分页（内存压力） 
■ 内存回收：等待内存回收例程

从技术上讲，调度程序延迟统计信息源自 schedstats（前面提到的 /proc），但与其他延迟记帐状态一起公开。（它在 struct sched_info 中，而不是 struct task_delay_info。

用户级工具可以使用 taskstats 读取这些统计信息，taskstats 是一个基于 netlink 的界面，用于获取每个任务和进程的统计信息。在内核源码中有：
■ Documentation/accounting/delay-accounting.txt： 文档 
■ tools/accounting/getdelays.c： 一个示例消费者

以下是 getdelays.c 的一些输出：

# $ ./getdelays -dp 17451
 print delayacct stats ON
 PID    17451
 CPU             count     real total  virtual total    delay total  delay average
                  386     3452475144    31387115236     1253300657          3.247ms
 IO              count    delay total  delay average
                  302     1535758266              5ms
 SWAP            count    delay total  delay average
                    0              0              0ms
 RECLAIM         count    delay total  delay average
                    0              0              0ms

除非另有说明，否则时间以纳秒为单位。此示例取自 CPU 负载较重的系统，并且检查的进程存在调度程序延迟。

# #4.3.4 netlink
netlink 是一个特殊的套接字地址族 （AF_NETLINK） 用于获取内核信息。使用 netlink 涉及使用 AF_NETLINK 地址族打开一个网络套接字，然后使用一系列 send（2） 和 recv（2） 调用来传递请求和接收二进制结构体中的信息。虽然这是一个比 /proc 更复杂的接口，但它效率更高，并且还支持通知。libnetlink 库有助于使用。

与早期工具一样，strace（1） 可用于显示内核信息的来源。检查套接字统计工具 ss（8）：

 # strace ss
 [...]
 socket(AF_NETLINK, SOCK_RAW|SOCK_CLOEXEC, NETLINK_SOCK_DIAG) = 3
 [...]

这将为组 NETLINK_SOCK_DIAG 打开一个 AF_NETLINK 套接字，该套接字返回有关套接字的信息。它记录在 sock_diag（7） 手册页中。NetLink 组包括：

■ NETLINK_ROUTE：路由信息（还有 /proc/net/route） 
■ NETLINK_SOCK_DIAG：套接字信息 
■ NETLINK_SELINUX：SELinux 事件通知 
■ NETLINK_AUDIT：审计（安全） 
■ NETLINK_SCSITRANSPORT：SCSI 传输 NETLINK_CRYPTO：内核加密信息

使用 netlink 的命令包括 ip（8）、ss（8）、routel（8） 以及较旧的 ifconfig（8） 和 netstat（8）。

# #4.3.5 追踪点
跟踪点是基于静态检测的 Linux 内核事件源，该术语在第 1 章 引言， 第 1.7.3 节 跟踪中介绍。跟踪点是放置在内核代码中逻辑位置的硬编码插桩点。例如，在系统调用、计划程序事件、文件系统作和磁盘 I/O 的开始和结束时都有跟踪点。4 跟踪点基础结构由 Mathieu Desnoyers 开发，并在 2009 年的 Linux 2.6.32 版本中首次提供。跟踪点是一个稳定的 API5，数量有限。

跟踪点是性能分析的重要资源，因为它们为超越摘要统计信息的高级跟踪工具提供支持，从而更深入地了解内核行为。虽然基于函数的跟踪可以提供类似的功能（例如，第 4.3.6 节，kprobes），但只有跟踪点提供稳定的接口，从而允许开发强大的工具。

本节介绍跟踪点。部分 4.5， 跟踪工具中介绍的跟踪程序可以使用这些跟踪程序，第 13 章到第 15 章将对此进行深入介绍。

# 跟踪点示例
可以使用 perf list 命令列出可用的跟踪点（perf（1） 语法的语法在第 14 章中介绍）：

# perf list tracepoint
 List of pre-defined events (to be used in -e):
 [...]
  block:block_rq_complete                            [Tracepoint event]
  block:block_rq_insert                              [Tracepoint event]
  block:block_rq_issue                               [Tracepoint event]
 [...]
  sched:sched_wakeup                                 [Tracepoint event]
  sched:sched_wakeup_new                             [Tracepoint event]
  sched:sched_waking                                 [Tracepoint event]
  scsi:scsi_dispatch_cmd_done                        [Tracepoint event]
  scsi:scsi_dispatch_cmd_error                       [Tracepoint event]
  scsi:scsi_dispatch_cmd_start                       [Tracepoint event]
  scsi:scsi_dispatch_cmd_timeout                     [Tracepoint event]
 [...]
  skb:consume_skb                                    [Tracepoint event]
  skb:kfree_skb                                      [Tracepoint event]
 [...]

我截断了输出，以显示来自块设备层、计划程序和 SCSI 的十几个示例跟踪点。在我的系统上，有 1808 个不同的跟踪点，其中 634 个用于检测系统调用。

除了显示事件发生的时间外，跟踪点还可以提供有关事件的上下文数据。例如，下面的 perf（1） 命令跟踪 block：block_rq_issue 跟踪点并实时打印事件：

 # perf trace -e block:block_rq_issue
 [...]
     0.000 kworker/u4:1-e/20962 block:block_rq_issue:259,0 W 8192 () 875216 + 16 
[kworker/u4:1]
   255.945 :22696/22696 block:block_rq_issue:259,0 RA 4096 () 4459152 + 8 [bash]
   256.957 :22705/22705 block:block_rq_issue:259,0 RA 16384 () 367936 + 32 [bash]
 [...]

前三个字段是时间戳（秒）、进程详细信息（名称/线程 ID）和事件描述（后跟冒号分隔符而不是空格）。其余字段是跟踪点的参数，由下面介绍的格式字符串生成;有关特定的 block：block_rq_issue 格式字符串，请参阅第 9 章 磁盘， 第 9.6.5 节 perf。

关于术语的说明：tracepoints （或 trace points） 在技术上是放置在内核源代码中的跟踪函数（也称为跟踪钩子）。例如，trace_sched_wakeup（） 是一个跟踪点，你会发现它是从 kernel/sched/core.c 调用的。此跟踪点可以通过使用名称 “sched：sched_wakeup” 的跟踪器进行检测;但是，从技术上讲，这是一个由 TRACE_EVENT 宏定义的跟踪事件。TRACE_EVENT 还定义和格式化其参数，自动生成 trace_sched_wakeup（） 代码，并将跟踪事件放在 tracefs 和 perf_event_open（2） 接口 [Ts'o 20] 中。跟踪工具主要检测跟踪事件，尽管它们可能将它们称为 “tracepoint”。perf（1） 将跟踪事件称为 “Tracepoint event”，这很令人困惑，因为基于 kprobe 和 uprobe 的跟踪事件也被标记为 “Tracepoint event”。

# 跟踪点参数和格式字符串
每个跟踪点都有一个格式字符串，其中包含事件参数：有关事件的额外上下文。此格式字符串的结构可以在 /sys/kernel/debug/ tracing/events 下的“format”文件中看到。例如：

 # cat /sys/kernel/debug/tracing/events/block/block_rq_issue/format 
name: block_rq_issue
 ID: 1080
 format:
        field:unsigned short common_type;  offset:0;  size:2;  signed:0;
        field:unsigned char common_flags;  offset:2;  size:1;  signed:0;
        field:unsigned char common_preempt_count;  offset:3;  size:1;  signed:0;
        field:int common_pid;   offset:4;  size:4;  signed:1;
        field:dev_t dev;        offset:8;  size:4;  signed:0;
        field:sector_t sector;  offset:16;  size:8;  signed:0;
        field:unsigned int nr_sector;  offset:24;  size:4;  signed:0;
        field:unsigned int bytes;      offset:28;  size:4;  signed:0;
        field:char rwbs[8];   offset:32;  size:8;  signed:1;
        field:char comm[16];  offset:40;  size:16;  signed:1;
        field:__data_loc char[] cmd;  offset:56;  size:4;  signed:1;
 print fmt: "%d,%d %s %u (%s) %llu + %u [%s]", ((unsigned int) ((REC->dev) >> 20)), 
((unsigned int) ((REC->dev) & ((1U << 20) - 1))), REC->rwbs, REC->bytes, 
__get_str(cmd), (unsigned long long)REC->sector, REC->nr_sector, REC->comm

最后一行显示字符串格式和参数。下面显示了此输出中的格式字符串格式，后跟上一个 perf 脚本输出中的示例格式字符串：

 %d,%d %s %u (%s) %llu + %u [%s]
 259,0 W 8192 () 875216 + 16 [kworker/u4:1]

这些匹配

Tracer 通常可以通过名称访问格式字符串中的参数。例如，以下命令仅在 size（bytes 参数）大于 65536时使用 perf（1） 来跟踪块 I/O 问题事件：

 # perf trace -e block:block_rq_issue --filter 'bytes > 65536'
     0.000 jbd2/nvme0n1p1/174 block:block_rq_issue:259,0 WS 77824 () 2192856 + 152 
[jbd2/nvme0n1p1-]
     5.784 jbd2/nvme0n1p1/174 block:block_rq_issue:259,0 WS 94208 () 2193152 + 184 
[jbd2/nvme0n1p1-]
 [...]

作为不同跟踪器的示例，以下使用 bpftrace 仅打印此跟踪点的 bytes 参数（bpftrace 语法在第 15 章 BPF 中介绍;我将在后续示例中使用 bpftrace，因为它使用起来很简洁，需要的命令更少）：

# bpftrace -e 't:block:block_rq_issue { printf("size: %d bytes\n", args->bytes); }'
 Attaching 1 probe...
 size: 4096 bytes
 size: 49152 bytes
 size: 40960 bytes
 [...]

输出为每个 I/O 问题一行，显示其大小。跟踪点是一个稳定的 API，由跟踪点名称、格式字符串和参数组成。

# Tracepoints 接口
跟踪工具可以通过 tracefs（通常挂载在 /sys/kernel/debug/tracing 中）或 perf_event_open（2） syscall 中的跟踪事件文件来使用跟踪点。例如，我的基于 Ftrace 的 iosnoop（8） 工具使用 tracefs 文件：

 # strace -e openat ~/Git/perf-tools/bin/iosnoop
 chdir("/sys/kernel/debug/tracing")      = 0
 openat(AT_FDCWD, "/var/tmp/.ftrace-lock", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
 [...]
 openat(AT_FDCWD, "events/block/block_rq_issue/enable", O_WRONLY|O_CREAT|O_TRUNC, 
0666) = 3
 openat(AT_FDCWD, "events/block/block_rq_complete/enable", O_WRONLY|O_CREAT|O_TRUNC, 
0666) = 3
 [...]

输出包括 tracefs 目录的 chdir（2） 和打开块跟踪点的 “enable” 文件。它还包括一个 /var/tmp/.ftrace-lock：这是我编写的预防措施，它会阻止当前的工具用户，而 tracefs 接口并不容易支持。perf_event_open（2） 接口支持并发用户，并且尽可能使用。它由同一工具的较新 BCC 版本使用：

 # strace -e perf_event_open /usr/share/bcc/tools/biosnoop 
perf_event_open({type=PERF_TYPE_TRACEPOINT, size=0 /* PERF_ATTR_SIZE_??? */, 
config=2323, ...}, -1, 0, -1, PERF_FLAG_FD_CLOEXEC) = 8
 perf_event_open({type=PERF_TYPE_TRACEPOINT, size=0 /* PERF_ATTR_SIZE_??? */, 
config=2324, ...}, -1, 0, -1, PERF_FLAG_FD_CLOEXEC) = 10
 [...]

perf_event_open（2） 是内核perf_events子系统的接口，它提供各种分析和跟踪功能。有关更多详细信息，请参阅其手册页，以及第 13 章中的 perf（1） 前端。

# 跟踪点开销
激活跟踪点后，它们会为每个事件增加少量 CPU 开销。跟踪工具还可能会增加后处理事件的 CPU 开销，以及记录事件的文件系统开销。开销是否高到足以扰乱生产应用程序取决于事件的速率和 CPU 的数量，这是您在使用跟踪点时需要考虑的问题。

在当今的典型系统（4 到 128 个 CPU）上，我发现每秒小于 10000 的事件速率开销可以忽略不计，只有超过 100000 个开销才开始变得可衡量。作为事件示例，您可能会发现磁盘事件通常少于每秒 10,000 个，但计划程序事件可能远高于每秒 100,000 个，因此跟踪成本可能很高。

我之前分析过特定系统的开销，发现最小跟踪点开销是 96 纳秒的 CPU 时间 [Gregg 19]。Linux 4.7 中添加了一种称为原始跟踪点的新型跟踪点，该跟踪点于 2018 年添加到Linux中，它避免了创建稳定跟踪点参数的成本，从而减少了此开销。

除了在使用跟踪点时启用的开销外，还有使跟踪点可用的禁用开销。禁用的跟踪点将变为少量指令：x86_64它是一个 5 字节的无作 （nop） 指令。函数末尾还添加了一个跟踪点处理程序，这会稍微增加其文本大小。虽然这些开销非常小，但在向内核添加跟踪点时，您应该分析和了解它们。

# Tracepoint 文档
跟踪点技术记录在内核源代码的 Documentation/trace/ tracepoints.rst 下。跟踪点本身（有时）记录在定义它们的头文件中，可在 Linux 源的 include/trace/events 下找到。我总结高级BPF 性能工具第 2 章 [Gregg 19] 中的跟踪点主题：如何将它们添加到内核代码中，以及它们如何在指令级别工作

有时您可能希望跟踪没有跟踪点的软件执行：为此，您可以尝试不稳定的 kprobes 接口。

# #4.3.6 内核函数
kprobes（内核探测的缩写）是基于动态检测的跟踪程序的 Linux 内核事件源，该术语在第 1 章 引言， 第 1.7.3 节 跟踪中介绍。kprobes 可以跟踪任何内核功能或指令，并在 2004 年发布的 Linux 2.6.9 中提供。它们被认为是不稳定的 API，因为它们公开了原始内核函数和参数，这些函数和参数可能会因内核版本而异。

kprobes 可以在内部以不同的方式工作。标准方法是修改正在运行的内核代码的指令文本，以在需要时插入 instrumentation。当 instrumenting 函数的入口时，如果 kprobes 使用现有的 Ftrace 函数跟踪，则可以使用优化，因为它的开销较低。

kprobes 可以在内部以不同的方式工作。标准方法是修改正在运行的内核代码的指令文本，以在需要时插入 instrumentation。当 instrumenting 函数的入口时，如果 kprobes 使用现有的 Ftrace 函数跟踪，则可以使用优化，因为它的开销较低。

kprobes 和 tracepoints 在表 4.3 中进行了比较

kprobes 可以跟踪函数的入口以及函数内的指令偏移量。使用 kprobes 会创建 kprobe 事件（基于 kprobe 的跟踪事件）。这些 kprobe 事件仅在跟踪器创建时存在：默认情况下，内核代码运行不加修改。

# kprobes 示例
作为使用 kprobes 的示例，以下 bpftrace 命令检测 do_ nanosleep（） 内核函数并打印 CPU 上的进程

 # bpftrace -e 'kprobe:do_nanosleep { printf("sleep by: %s\n", comm); }'
 Attaching 1 probe...
 sleep by: mysqld
 sleep by: mysqld
 sleep by: sleep
 ^C
 #
输出显示名为 “mysqld” 的进程的几个休眠，以及 “sleep” 的一个休眠（可能是 /bin/sleep）。do_nanosleep（） 的 kprobe 事件在 bpftrace 程序开始运行时创建，并在 bpftrace 终止时删除 （Ctrl-C）。

# kprobes 参数
由于 kprobes 可以跟踪内核函数调用，因此通常需要检查函数的参数以获取更多上下文。每个跟踪工具都以自己的方式公开它们，并在后面的部分中介绍。例如，使用 bpftrace 将第二个参数打印到 do_nanosleep（），即 hrtimer_mode：

 # bpftrace -e 'kprobe:do_nanosleep { printf("mode: %d\n", arg1); }'
 Attaching 1 probe...
 mode: 1
 mode: 1
 mode: 1
 [...]

函数参数在 bpftrace 中使用 arg0..argN 内置变量

# kretprobes
内核函数的返回值及其返回值可以使用 kretprobes（内核返回探针的缩写）进行跟踪，这与 kprobes 类似。KretProbes 是使用函数 Entry 的 Kprobe 实现的，该函数插入一个 trampoline 函数来检测返回值。

当与 kprobes 和记录时间戳的跟踪器配对时，可以测量内核功能的持续时间。例如，使用 bpftrace 测量 do_nanosleep（） 的持续时间

 # bpftrace -e 'kprobe:do_nanosleep { @ts[tid] = nsecs; }
    kretprobe:do_nanosleep /@ts[tid]/ {
    @sleep_ms = hist((nsecs - @ts[tid]) / 1000000); delete(@ts[tid]); }
    END { clear(@ts); }'

     @sleep_ms: 
    [0]                 1280 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
    [1]                    1 |                                                    |
    [2, 4)                 1 |                                                    |
    [4, 8)                 0 |                                                    |
    [8, 16)                0 |                                                    |
    [16, 32)               0 |                                                    |
    [32, 64)               0 |                                                    |
    [64, 128)              0 |                                                    |
    [128, 256)             0 |                                                    |
    [256, 512)             0 |                                                    |
    [512, 1K)              2 |                                                    |

输出显示 do_nanosleep（） 通常是一个快速函数，以零毫秒（四舍五入）返回 1,280 次。两次出现达到 512 到 1024 毫秒的范围。

bpftrace 语法在第 15 章 BPF 中进行了解释，其中包括一个类似的计时 vfs_read（） 示例

# kprobes 接口和开销
kprobes 接口类似于 tracepoint。有一种方法可以通过 /sys 文件、perf_event_open（2） syscall（首选）以及 register_kprobe（） 内核 API 来检测它们。当跟踪函数的条目时（Ftrace 方法，如果可用），开销类似于跟踪点的开销，当跟踪函数偏移量（断点方法）或使用 kretprobes 时（trampoline 方法）的开销更高。对于特定系统，我测量的最小 kprobe CPU 成本为 76 纳秒，最小 kretprobe CPU 成本为 212 纳秒 [Gregg 19]

# kprobes 文档
kprobe 记录在 Linux 源代码的 Documentation/kprobes.txt 下。他们检测的内核函数通常不会在内核源之外进行记录（因为大多数不是 API，所以不需要）。我在 BPF 性能工具，第 2 章 [Gregg 19]：它们在指令级别是如何工作的。

# #4.3.7 uprobes
uprobes（用户空间探测器）类似于 kprobes，但适用于用户空间。它们可以动态检测应用程序和库中的函数，并提供不稳定的 API，用于深入研究其他工具范围之外的软件内部结构。uprobe 在 2012 年发布的 Linux 3.5 中可用。

部分 4.5 “跟踪工具”中介绍的跟踪器可以使用 uprobe，并在第 13 章到第 15 章中进行了深入介绍。

# uprobes 示例
作为 uprobe 的一个示例，以下 bpftrace 命令列出了 bash（1） shell 中可能的 uprobe 函数入口位置：
 # bpftrace -l 'uprobe:/bin/bash:*'
 uprobe:/bin/bash:rl_old_menu_complete
 uprobe:/bin/bash:maybe_make_export_env
 uprobe:/bin/bash:initialize_shell_builtins
 uprobe:/bin/bash:extglob_pattern_p
 uprobe:/bin/bash:dispose_cond_node
 uprobe:/bin/bash:decode_prompt_string
 [..]

完整输出显示 1,507 个可能的 uprobe。uprobes 检测代码并在需要时创建 uprobe 事件（基于 uprobe 的跟踪事件）：默认情况下，用户空间代码不加修改地运行。这类似于使用调试器向函数添加断点：在添加断点之前，函数将运行而不修改。

# uprobes 参数
用户函数的参数由 uprobe 提供。例如，以下代码使用 bpftrace 检测 decode_prompt_string（） bash 函数并将第一个参数打印为字符串：

# bpftrace -e 'uprobe:/bin/bash:decode_prompt_string { printf("%s\n", str(arg0)); }'
 Attaching 1 probe...
 \[\e[31;1m\]\u@\h:\w>\[\e[0m\] 
\[\e[31;1m\]\u@\h:\w>\[\e[0m\] 
^C

输出显示此系统上的 bash（1） 提示符字符串。decode_prompt_ string（） 的 uprobe 是在 bpftrace 程序开始运行时创建的，并在 bpftrace 终止时删除 （Ctrl-C）

# uretprobes
用户函数的返回值及其返回值可以使用 uretprobes（用户级返回探针的缩写）进行跟踪，这与 uprobes 类似。当与 uprobe 和记录时间戳的跟踪器配对时，可以测量用户级函数的持续时间。请注意，uretprobes 的开销会显著偏斜这种快速函数的测量

# uprobes 接口和开销
uprobes 接口类似于 kprobes。有一种方法可以通过 /sys 文件和（最好）通过 perf_event_open（2） syscall 来检测它们。uprobe 目前通过捕获到内核中来工作。这比 kprobe 或 tracepoint 消耗的 CPU 开销高得多。对于我测量的特定系统，最小 uprobe 成本为 1,287 纳秒，最小 uretprobe 成本为 1,931 纳秒 [Gregg 19]。uretprobe 开销更高，因为它是一个 uprobe 加一个 trampoline 函数。

# uprobe 文档
uprobe 记录在 Linux 源代码的 Documentation/trace/uprobetracer.rst 下。我在 BPF Performance Tools， Chapter 2 [Gregg 19]： how they work at the instruction level 中总结了高级 uprobe 主题。他们检测的用户函数通常不会在应用程序源之外进行文档化（因为大多数函数不太可能是 API，因此不需要是 API）。对于记录在案的用户空间跟踪，请使用 USDT。

# #4.3.8 USDT
用户级静态定义跟踪 （USDT） 是跟踪点的用户空间版本。USDT 之于 uprobes，就像 tracepoints 之于 kprobes。一些应用程序和库已将 USDT 探针添加到其代码中，为跟踪应用程序级事件提供了稳定（且有文档记录的）API。例如，Java JDK、PostgreSQL 数据库和 libc 中都有 USDT 探针。下面列出了使用 bpftrace 的 OpenJDK USDT 探针

# bpftrace -lv 'usdt:/usr/lib/jvm/openjdk/libjvm.so:*'
 usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:class__loaded
 usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:class__unloaded
 usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:method__compile__begin
 usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:method__compile__end
 usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:gc__begin
 usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:gc__end
 [...]

其中列出了用于 Java 类加载和卸载、方法编译和垃圾回收的 USDT 探测器。还有更多被截断：完整列表显示此 JDK 版本的 524 USDT 探针。

许多应用程序已经具有可以启用和配置的自定义事件日志，并且对于性能分析非常有用。USDT 探针的不同之处在于，它们可以从各种跟踪器中使用，这些跟踪器可以将应用程序上下文与内核事件（如磁盘和网络 I/O）相结合。应用程序级记录器可能会告诉您，由于文件系统 I/O 导致数据库查询速度变慢，但跟踪器可以揭示更多信息：例如，查询速度变慢是由于文件系统中的锁定限制，而不是您可能假设的磁盘 I/O

某些应用程序包含 USDT 探针，但当前未在应用程序的打包版本中启用它们（OpenJDK 就是这种情况）。使用它们需要使用适当的 config 选项从源重新构建应用程序。该选项可以在 DTrace 跟踪器之后称为 --enable-dtrace-probes，DTrace 跟踪器推动了 USDT 在应用程序中的采用。

USDT 探针必须编译到它们检测的可执行文件中。这对于 JIT 编译的语言（如 Java）来说是不可能的，因为 Java 通常是动态编译的。解决这个问题的办法是dynamic USDT，它将探针预编译为共享库，并提供一个接口，以便从 JIT 编译的语言调用它们。存在适用于 Java、Node.js 和其他语言的动态 USDT 库。解释型语言也有类似的问题，需要动态 USDT。

USDT 探针是在 Linux 中使用 uprobe 实现的：有关 uprobe 及其开销的描述，请参阅上一节。除了启用的开销之外，USDT 探针还会在代码中放置 nop 指令，跟踪点也是如此。

第 4.5 节 “跟踪工具”中介绍的跟踪器可以使用 USDT 探针，第 13 章到第 15 章将对此进行了深入介绍（尽管将 USDT 与 Ftrace 一起使用需要一些额外的工作）。

# USDT 文档
如果应用程序提供 USDT 探针，则应将其记录在应用程序的文档中。我在 BPF 性能工具第 2 章 [Gregg 19] 中总结了高级 USDT 主题：如何将 USDT 探针添加到应用程序代码中，它们如何在内部工作，以及动态 USDT。

# #4.3.9 硬件计数器（PMCs）
处理器和其他设备通常支持用于观察活动的硬件计数器。主要来源是处理器，它们通常称为性能监控计数器 （PMC）。它们还有其他名称：CPU 性能计数器 （CPC）、性能检测计数器 （PIC） 和性能监控单元事件 （PMU 事件）。这些都是指同一件事： 处理器上的可编程硬件寄存器，它在 CPU cycle 级别提供低级性能信息。

PMC 是性能分析的重要资源。只有通过 PMC 才能测量 CPU 指令的效率、CPU 缓存的命中率、内存和设备总线的利用率、互连利用率、停顿周期等。使用这些来分析性能可以实现各种性能优化。

# PMC 示例
虽然有许多 PMC，但 Intel 选择了 7 个作为“架构集”，它们提供了一些核心功能的高级概述 [Intel 16]。可以使用 cpuid 指令检查这些架构集 PMC 的存在。表 4.4 显示了这组 PMC，它作为一组有用的 PMC 示例

以 PMC 为例，如果在未指定事件（无 -e）的情况下运行 perf stat 命令，则默认为检测架构 PMC。例如，以下对 gzip（1） 命令运行 perf stat：

 # perf stat gzip words
 Performance counter stats for 'gzip words':
        156.927428      task-clock (msec)         #    0.987 CPUs utilized          
                 1      context-switches          #    0.006 K/sec                  
                 0      cpu-migrations            #    0.000 K/sec                  
               131      page-faults               #    0.835 K/sec                  
       209,911,358      cycles                    #    1.338 GHz                    
       288,321,441      instructions              #    1.37  insn per cycle         
        66,240,624      branches                  #  422.110 M/sec                  
         1,382,627      branch-misses             #    2.09% of all branches        
       0.159065542 seconds time elapsed

原始计数是第一列;哈希之后是一些统计数据，包括一个重要的性能指标：每个周期的指令数 （INNS per cycle）。这表明 CPU 执行指令的效率 — 越高越好。此指标在第 6 章 CPU 的第 6.3.7 节 IPC 中进行了解释。

# PMC 接口
在 Linux 上，PMC 通过 perf_event_open（2） syscall 访问，并被包括 perf（1） 在内的工具使用。

虽然有数百个 PMC 可用，但 CPU 中只有固定数量的寄存器可用于同时测量它们，可能只有 6 个。你需要选择你想在这 6 个寄存器上测量的 PMC，或者循环使用不同的 PMC 集作为采样的一种方式（Linux perf（1） 自动支持此功能）。其他软件计数器不受这些约束。

PMC 可以在不同的模式下使用：计数，它们以几乎为零的开销对事件进行计数;以及溢出采样，其中在每个可配置的事件数量中引发一个中断，以便可以捕获状态。计数可用于量化问题;overflow sampling 可用于显示负责的代码路径。

perf（1） 可以使用 stat 子命令执行计数，并使用 record 子命令执行采样;见第 13 章，性能。

# PMC 挑战
使用 PMC 时的两个常见挑战是它们对溢出采样的准确性以及它们在云环境中的可用性。

由于中断延迟（通常称为“滑橇”）或无序指令执行，溢出采样可能无法记录触发事件的正确指令指针。对于 CPU 周期分析，这种撬块可能不是问题，一些分析器故意引入抖动以避免锁步采样（或使用偏移采样率，如 99 赫兹）。但是为了测量其他事件，例如 LLC 未命中，采样的指令指针需要准确。

解决方案是处理器对所谓的精确事件的支持。在 Intel 上，精确事件使用一种称为基于事件的精确采样 （PEBS） 的技术，9 该技术使用硬件缓冲区在 PMC 事件发生时记录更准确（“精确”）的指令指针。在 AMD 上，精确事件使用基于指令的采样 （IBS） [Drongowski 07]。Linux perf（1） 组件支持精确事件（参见第 13 章 perf， Section 13.9.2 CPU 性能分析）

另一个挑战是云计算，因为许多云环境禁止其访客访问 PMC。从技术上讲，启用它是可能的：例如，Xen 虚拟机管理程序具有 vpmu 命令行选项，它允许向 guest10 [Xenbits 20] 公开不同的 PMC 集。Amazon 已经为其 Nitro 管理程序 Guest 启用了许多 PMC。11 此外，一些云提供商提供“裸机实例”，其中 Guest 具有完全的处理器访问权限，因此具有完全的 PMC 访问权限

# PMC 文档
PMC 是特定于处理器的，并记录在相应的处理器软件开发人员手册中。处理器制造商示例：
■ 英特尔：英特尔® 64 位和 IA-32 架构软件开发人员手册第 3 卷 [英特尔 16] 的第 19 章“性能监控事件”。
■ AMD：AMD 系列 17h 处理器型号 00h-2Fh 开源寄存器参考的第 2.1.1 节“性能监视器计数器”[AMD 18]
■ ARM：Arm® 体系结构参考手册 Armv8 的第 D7.10 节“PMU 事件和事件编号”，适用于 Armv8-A 体系结构配置文件 [ARM 19]

已经有人为所有处理器开发的 PMC 标准命名方案，称为性能应用程序编程接口 （PAPI） [UTK 20]。作系统对 PAPI 的支持喜忧参半：它需要频繁更新以将 PAPI 名称映射到供应商 PMC 代码。

第 6 章 CPU 的第 6.4.1 节 硬件的硬件计数器 （PMC） 小节更详细地介绍了它们的实现，并提供了其他 PMC 示例。

# #4.3.10 其他可观测资源
其他可观测性来源包括： 
■ MSR：PMC 使用特定于模型的寄存器 （MSR） 实现。还有其他 MSR 用于显示系统的配置和运行状况，包括 CPU 时钟速率、使用情况、温度和功耗。可用的 MSR 取决于处理器类型（特定于型号）、BIOS 版本和设置以及虚拟机管理程序设置。一种用途是基于周期的 CPU 利用率的准确测量。
■ ptrace（2）：此 syscall 控制进程跟踪，gdb（1） 用于进程调试，strace（1） 用于跟踪 syscall。它是基于断点的，可以将目标的速度减慢 100 倍以上。Linux 也有跟踪点，在第 4.3.5 节 跟踪点中介绍，用于更高效的系统调用跟踪。
■ 函数分析：分析函数调用（mcount（） 或 __fentry__（）））被添加到 x86 上所有非内联内核函数的开头，以实现高效的 Ftrace 函数跟踪。它们被转换为 nop 指令，直到需要为止。请参见第 14 章 “Ftrace”
■ 网络嗅探 （libpcap）：这些接口提供了一种从网络工作设备捕获数据包的方法，以便详细调查数据包和协议性能。在 Linux 上，嗅探是通过 libpcap 库和 /proc/net/dev 提供的，并由 tcpdump（8） 工具使用。捕获和检查所有数据包会产生 CPU 和存储开销。有关网络嗅探的更多信息，请参阅第 10 章
■ netfilter conntrack：Linux netfilter 技术允许在事件上执行自定义处理程序，不仅适用于防火墙，还用于连接跟踪 （conntrack）。这允许创建网络流的日志 [Ayuso 12]。
■ 流程会计：这可以追溯到大型机，需要根据流程的执行和运行时间向部门和用户收取计算机使用费。它以某种形式存在于 Linux 和其他系统中，有时有助于进程级别的性能分析。例如，Linux atop（1） 工具使用进程记帐来捕获和显示来自短期进程的信息，否则在拍摄 /proc [Atoptool 20] 的快照时会错过这些信息。
■ 软件事件：这些事件与硬件事件相关，但在软件中检测。页面错误就是一个例子。软件事件可通过 perf_event_open（2） 接口获得，并由 perf（1） 和 bpftrace 使用。它们如图 4.5 所示。
■ 系统调用：某些系统或库调用可能可用于提供某些性能指标。其中包括 getrusage（2），这是一个系统调用，用于进程获取自己的资源使用统计信息，包括用户和系统时间、故障、消息和上下文切换。

如果您对这些接口的工作原理感兴趣，您将发现通常提供文档，这些文档适用于在这些接口上构建工具的开发人员。

# 以及更多
根据您的内核版本和启用的选项，可能会有更多的可观测性源可用。有些在本书的后面几章中提到。对于 Linux，这些包括 I/O accounting、blktrace、timer_stats、lockstat 和 debugfs

找到此类源的一种方法是阅读您有兴趣观察的内核代码，并查看那里放置了哪些统计数据或跟踪点。

在某些情况下，可能没有关于您所追求的内容的内核统计数据。除了动态工具（Linux kprobes 和 uprobes）之外，你可能会发现像 gdb（1） 和 lldb（1） 这样的调试器可以获取内核和应用程序变量来为调查提供一些信息。

# Solaris Kstat
作为提供系统统计信息的不同方法的一个示例，基于 Solaris 的系统使用内核统计信息 （Kstat） 框架，该框架提供一致的内核统计信息分层结构，每个结构使用以下四元组命名：

 module:instance:name:statistic
这些模块是
■ 模块：这通常是指创建统计信息的内核模块，例如用于 SCSI 磁盘驱动程序的 sd，或用于 ZFS 文件系统的 zfs。
■ instance：某些模块以多个实例的形式存在，例如每个 SCSI 磁盘对应一个 sd 模块。该实例是一个枚举。
■ name：这是统计信息组的名称。
■ statistic：这是单个统计数据名称。

Kstats 使用二进制内核接口进行访问，并且存在各种库。

作为 Kstat 示例，以下代码使用 kstat（1M） 读取 “nproc” 统计信息并指定完整的四元组

# $ kstat -p unix:0:system_misc:nproc
 unix:0:system_misc:nproc        94

此统计信息显示当前正在运行的进程数。

相比之下，Linux 上的 /proc/stat-style 源的格式不一致，通常需要文本解析才能处理，这会消耗一些 CPU 周期。

# 4.4 sar
sar（1） 是在 第 4.2.4 节 监控 中引入的，作为一个关键的监控工具。虽然最近 BPF 跟踪超级大国令人兴奋不已（我对此负有部分责任），但您不应忽视 sar（1） 的实用性 — 它是一个必不可少的系统性能工具，可以自行解决许多性能问题。Linux 版本的 sar（1） 也设计精良，具有自描述的列标题、网络度量组和详细的文档（手册页）。

sar（1） 通过 sysstat 软件包提供

# #4.4.1 sar 覆盖率
图 4.6 显示了不同 sar（1） 命令行选项的可观测性覆盖率

该图显示 sar（1） 提供了对内核和设备的广泛覆盖，甚至对粉丝具有可观察性。-m （电源管理）选项还支持此图中未显示的其他参数，包括用于电压输入的 IN、用于设备温度的 TEMP 和用于 USB 设备功率统计的 USB。

# #4.4.2 sar 监控
您可能会发现 Linux 系统已经启用了 sar（1） 数据收集（监控）。如果不是，则需要启用它。要检查，只需运行 sar 而不带选项。例如：

# $ sar
 Cannot open /var/log/sysstat/sa19: No such file or directory
 Please check if data collecting is enabled

输出显示 sar（1） 数据收集尚未在此系统上启用（sa19 文件是指每月 19 日的每日存档）。启用它的步骤可能因您的分配而异。

# 配置 （Ubuntu）
在这个 Ubuntu 系统上，我可以通过编辑 /etc/default/sysstat 文件并将 ENABLED 设置为 true 来启用 sar（1） 数据收集：

 ubuntu# vi /etc/default/sysstat 
#
 # Default settings for /etc/init.d/sysstat, /etc/cron.d/sysstat
 # and /etc/cron.daily/sysstat files
 #
 # Should sadc collect system activity informations? Valid values
 # are "true" and "false". Please do not put other values, they
 # will be overwritten by debconf!
 ENABLED="true"

然后使用

 # ubuntu# service sysstat restart

可以在 sysstat 的 crontab 文件中修改统计信息记录的计划：

 ubuntu# cat /etc/cron.d/sysstat 
# The first element of the path is a directory where the debian-sa1
 # script is located
 PATH=/usr/lib/sysstat:/usr/sbin:/usr/sbin:/usr/bin:/sbin:/bin
 # Activity reports every 10 minutes everyday
 5-55/10 * * * * root command -v debian-sa1 > /dev/null && debian-sa1 1 1
 # Additional run at 23:59 to rotate the statistics file
 59 23 * * * root command -v debian-sa1 > /dev/null && debian-sa1 60 2

语法 5-55/10 表示它将在每小时 5 到 55 分钟的分钟范围内每 10 分钟记录一次。您可以调整它以适应所需的分辨率：语法记录在 crontab（5） 手册页中。更频繁的数据收集将增加 sar（1） 存档文件的大小，可以在 /var/log/sysstat 中找到。

我经常将数据收集更改为

 */5 * * * * root command -v debian-sa1 > /dev/null && debian-sa1 1 1 -S ALL

*/5 将每五分钟记录一次，而 -S ALL 将记录所有统计信息。默认情况下，sar（1） 将记录大多数（但不是全部）统计组。-S ALL 选项用于记录所有统计组 — 它被传递给 sadc（1），并记录在 sadc（1） 的手册页中。还有一个扩展版本 -S XALL，它记录了统计数据的其他细分。

# 报告
sar（1） 可以与图 4.6 中所示的任何选项一起执行，以报告选定的统计组。可以指定多个选项。例如，以下报告 CPU 统计信息 （-u）、TCP （-n TCP） 和 TCP 错误 （-n ETCP）

# $ sar -u -n TCP,ETCP
 Linux 4.15.0-66-generic (bgregg)  01/19/2020         _x86_64_        (8 CPU)
 10:40:01 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
 10:45:01 AM     all      6.87      0.00      2.84      0.18      0.00     90.12
 10:50:01 AM     all      6.87      0.00      2.49      0.06      0.00     90.58
 [...]
 10:40:01 AM  active/s passive/s    iseg/s    oseg/s
 10:45:01 AM      0.16      0.00     10.98      9.27
 10:50:01 AM      0.20      0.00     10.40      8.93
 [...]
 10:40:01 AM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
 10:45:01 AM      0.04      0.02      0.46      0.00      0.03
 10:50:01 AM      0.03      0.02      0.53      0.00      0.03
 [...]

输出的第一行是系统摘要，显示内核类型和版本、主机名、日期、处理器体系结构和 CPU 数量

运行 sar -A 将转储所有统计信息。

# 输出格式
sysstat 软件包附带一个 sadf（1） 命令，用于查看不同格式的 sar（1） 统计数据，包括 JSON、SVG 和 CSV。以下示例以这些格式发出 TCP （-n TCP） 统计信息。

# JSON (-j):
JavaScript 对象表示法 （JSON） 可以很容易地被许多编程语言解析和导入，使其成为在 sar（1） 上构建其他软件时合适的输出格式。

# $ sadf -j -- -n TCP
 {"sysstat": {
  "hosts": [
    {
      "nodename": "bgregg",
      "sysname": "Linux",
      "release": "4.15.0-66-generic",
      "machine": "x86_64",
      "number-of-cpus": 8,
      "file-date": "2020-01-19",
      "file-utc-time": "18:40:01",
      "statistics": [
        {
          "timestamp": {"date": "2020-01-19", "time": "18:45:01", "utc": 1, 
"interval": 300},
          "network": {
            "net-tcp": {"active": 0.16, "passive": 0.00, "iseg": 10.98, "oseg": 9.27}
          }
        },
 [...]

您可以使用 jq（1） 工具在命令行中处理 JSON 输出。

# SVG (-g):
sadf（1） 可以发出可在 Web 浏览器中查看的可缩放矢量图形 （SVG） 文件。图 4.7 显示了一个示例。您可以使用此输出格式构建基本的控制面板

# CSV (-d):
逗号分隔值 （CSV） 格式旨在由数据库导入（并使用分号）：

# $ sadf -d -- -n TCP
 # hostname;interval;timestamp;active/s;passive/s;iseg/s;oseg/s
 bgregg;300;2020-01-19 18:45:01 UTC;0.16;0.00;10.98;9.27
 bgregg;299;2020-01-19 18:50:01 UTC;0.20;0.00;10.40;8.93
 bgregg;300;2020-01-19 18:55:01 UTC;0.12;0.00;9.27;8.07
 [...]

# #4.4.3 sar(1) Live
当使用间隔和可选 count 执行时，sar（1） 会执行实时报告。即使未启用数据收集，也可以使用此模式。

例如，显示间隔为 1 秒且计数为 5 的 TCP 统计信息：

# $ sar -n TCP 1 5
 Linux 4.15.0-66-generic (bgregg)  01/19/2020         _x86_64_        (8 CPU)
 03:09:04 PM  active/s passive/s    iseg/s    oseg/s
 03:09:05 PM      1.00      0.00     33.00     42.00
 03:09:06 PM      0.00      0.00    109.00     86.00
 03:09:07 PM      0.00      0.00    107.00     67.00
 03:09:08 PM      0.00      0.00    104.00    119.00
 03:09:09 PM      0.00      0.00     70.00     70.00
 Average:         0.20      0.00     84.60     76.80

数据收集的间隔很长，例如 5 分钟或 10 分钟，而实时报告允许您查看每秒的变化。

后面的章节包括实时 sar（1） 统计信息的各种示例

# #4.4.4 sar文档
sar（1） 手册页记录了各个统计信息，并在方括号中包含 SNMP 名称。例如：

 $ man sar
 [...]
               active/s
                     The number of times TCP connections have  made  a  direct
                     transition  to  the  SYN-SENT state from the CLOSED state
                     per second [tcpActiveOpens].
                passive/s
                     The number of times TCP connections have  made  a  direct
                     transition  to  the  SYN-RCVD state from the LISTEN state
                     per second [tcpPassiveOpens].
              iseg/s
 [...]
                     The total number of segments received per second, includ
                     ing  those  received  in  error  [tcpInSegs].  This count
                     includes segments received on currently established  con
       
sar（1） 的具体用途将在本书后面描述;见第 6 章至第 10 章。附录 C 是 sar（1） 选项和度量的总和。

# 4.5 追踪工具
Linux 跟踪工具使用前面描述的事件接口（tracepoint、kprobes、uprobes、USDT）进行高级性能分析。主要的跟踪工具包括：

■ perf（1）：官方 Linux 分析器。它非常适合 CPU 分析 （堆栈跟踪采样） 和 PMC 分析，并且可以检测其他事件，通常记录到输出文件进行后处理。
■ Ftrace：Linux 官方跟踪器，它是一个由不同跟踪实用程序组成的多功能工具。它适用于内核代码路径分析和资源受限的系统，因为它可以在没有依赖项的情况下使用。BPF（BCC、bpftrace）：扩展 BPF 在第 3 章作系统， 第 3.4.4 节 扩展 BPF 中介绍。它为高级跟踪工具提供支持，主要是 BCC 和 bpftrace。BCC 提供了强大的工具，而 bpftrace 为自定义单行程序和短程序提供了高级语言。
■ SystemTap：一种高级语言和跟踪器，具有许多 tapset（库），用于跟踪不同的目标 [Eigler 05][Sourceware 20]。它最近一直在开发一个 BPF 后端，我推荐它（参见 stapbpf（8） 手册页）。
■ LTTng：针对黑盒记录优化的示踪剂：以最佳方式记录许多事件以供以后分析 [LTTng 20]。

前三个示踪剂在第 13 章 perf 中介绍;第 14 章，Ftrace;和第 15 章，BPF。接下来的章节（第 5 章到第 12 章）包括这些跟踪器的各种用法，显示了要键入的命令以及如何解释输出。这种排序是经过深思熟虑的，首先关注用途和性能胜利，然后根据需要稍后更详细地介绍示踪剂。

在 Netflix，我使用 perf（1） 进行 CPU 分析，使用 Ftrace 进行内核代码挖掘，将 BCC/bpftrace 用于其他所有内容（内存、文件系统、磁盘、网络和应用程序跟踪）

# 4.6 观察可观测性
可观测性工具及其构建所依据的统计数据是在软件中实现的，所有软件都有可能出现错误。描述 软件的文档也是如此。以健康的怀疑态度看待任何对您来说陌生的统计数据，质疑它们的真正含义以及它们是否真的正确。

度量可能会受到以下任何问题的影响： 
■ 工具和度量有时是错误的。
■ 手册页并不总是正确的。
■ 可用指标可能不完整。
■ 可用的指标可能设计不佳且令人困惑。
■ 度量收集器（例如，解析工具输出的收集器）可能存在错误。
■ 度量处理（算法/电子表格）也可能引入错误。

当多个可观测性工具具有重叠的覆盖范围时，您可以使用它们相互交叉检查。理想情况下，他们将提供不同的 instrumentation 框架来检查框架中的错误。动态插桩对此特别有用，因为可以创建自定义工具来仔细检查指标。

另一种验证技术是应用已知的工作负载，然后检查观测能力工具是否与您预期的结果一致。这可能涉及使用微基准测试工具，这些工具报告自己的统计数据以供比较。

有时，出错的不是工具或统计信息，而是描述它的文档，包括手册页。软件可能已经发展，但文档没有更新。

实际上，您可能没有时间仔细检查您使用的每项绩效衡量标准，并且只有在遇到不寻常的结果或对公司至关重要的结果时才会这样做。即使您不仔细检查，意识到您没有并且您认为这些工具是正确的，这也是很有价值的。

指标也可能不完整。当面对大量的工具和指标时，可能很容易假设它们提供了完整和有效的覆盖率。情况往往并非如此：指标可能已由程序员添加以调试自己的代码，然后内置到可观测性工具中，而没有对实际客户需求进行太多研究。一些程序员可能根本没有向新的子系统添加任何 CODE。

缺少指标可能比存在不良指标更难识别。第 2 章 方法论 可以通过研究性能分析需要回答的问题来帮助您找到这些缺失的指标。

# 4.7 习题
回答以下有关可观测性工具的问题（您可能希望重新访问第 1 章中对其中一些术语的介绍）：

1. 列出一些静态性能工具。
2. 什么是性能剖析？
3. 为什么分析器使用 99 赫兹而不是 100 赫兹？
4. 什么是跟踪？
5. 什么是静态仪表？
6. 描述为什么动态插桩很重要。
7. tracepoints 和 kprobes 有什么区别？
8. 描述以下预期的 CPU 开销（低/中/高）
■ 磁盘 IOPS 计数器（如 iostat（1） 所示） 
■ 通过跟踪点或 kprobe 跟踪每个事件的磁盘 I/O 跟踪每个事件的上下文切换 （tracepoints/kprobes） 
■ 跟踪每个事件的进程执行 （execve（2）） （tracepoints/kprobes） 
■ 通过 uprobe 跟踪每个事件的 libc malloc（）
9. 描述为什么 PMC 对性能分析很有价值。
10. 给定一个可观测性工具，描述如何确定它使用的插桩源。

# 4.8 引用
 [Eigler 05] Eigler, F. Ch., et al. “Architecture of SystemTap: A Linux Trace/Probe Tool,” 
http://sourceware.org/systemtap/archpaper.pdf, 2005.
 [Drongowski 07] Drongowski, P., “Instruction-Based Sampling: A New Performance 
Analysis Technique for AMD Family 10h Processors,” AMD (Whitepaper), 2007.
 [Ayuso 12] Ayuso, P., “The Conntrack-Tools User Manual,” http://conntrack-tools.netfilter.org/
 manual.html, 2012.
 [Gregg 13c] Gregg, B., “Benchmarking Gone Wrong,” Surge 2013: Lightning Talks, 
https://www.youtube.com/watch?v=vm1GJMp0QN4#t=17m48s, 2013
  [Weisbecker 13] Weisbecker, F., “Status of Linux dynticks,” OSPERT, 
http://www.ertl.jp/~shinpei/conf/ospert13/slides/FredericWeisbecker.pdf, 2013.
 [Intel 16] Intel 64 and IA-32 Architectures Software Developer’s Manual Volume 3B: System 
Programming Guide, Part 2, September 2016, https://www.intel.com/content/www/us/en/ 
architecture-and-technology/64-ia-32-architectures-software-developer-vol-3b-part-2
manual.html, 2016.
 [AMD 18] Open-Source Register Reference for AMD Family 17h Processors Models 00h-2Fh, 
https://developer.amd.com/resources/developer-guides-manuals, 2018.
 [ARM 19] Arm® Architecture Reference Manual Armv8, for Armv8-A architecture profile, 
https://developer.arm.com/architectures/cpu-architecture/a-profile/docs?_
 ga=2.78191124.1893781712.1575908489-930650904.1559325573, 2019.
 [Gregg 19] Gregg, B., BPF Performance Tools: Linux System and Application Observability, 
Addison-Wesley, 2019.
 [Atoptool 20] “Atop,” www.atoptool.nl/index.php, accessed 2020.
 [Bowden 20] Bowden, T., Bauer, B., et al., “The /proc Filesystem,” Linux documentation, 
https://www.kernel.org/doc/html/latest/filesystems/proc.html, accessed 2020.
 [Gregg 20a] Gregg, B., “Linux Performance,” http://www.brendangregg.com/
 linuxperf.html, accessed 2020.
 [LTTng 20] “LTTng,” https://lttng.org, accessed 2020.
 [PCP 20] “Performance Co-Pilot,” https://pcp.io, accessed 2020.
 [Prometheus 20] “Exporters and Integrations,” https://prometheus.io/docs/instrumenting/
 exporters, accessed 2020.
 [Sourceware 20] “SystemTap,” https://sourceware.org/systemtap, accessed 2020.
 [Ts’o 20] Ts’o, T., Zefan, L., and Zanussi, T., “Event Tracing,” Linux documentation, 
https://www.kernel.org/doc/html/latest/trace/events.html, accessed 2020.
 [Xenbits 20] “Xen Hypervisor Command Line Options,” https://xenbits.xen.org/docs/
 4.11-testing/misc/xen-command-line.html, accessed 2020.
 [UTK 20] “Performance Application Programming Interface,” http://icl.cs.utk.edu/papi, 
accessed 2020.
