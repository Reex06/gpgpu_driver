# gpgpu-driver 项目学习指南
### —— 面向【嵌入式Linux驱动工程师 / BSP开发】岗位求职的完整学习方案

> 本文档基于你本机的项目源码（`/home/cxr/桌面/gpu_driver/gpgpu_driver`）逐文件阅读后写成，
> 所有代码行号引用都真实有效，可以直接点击跳转。
> 文档假设你的水平：**C 语言基础扎实、只学过一点字符设备驱动框架、没有完整驱动项目经验**。
> 遇到不熟的术语本文都会先解释再使用。

---

## 目录

- [0. 项目全景速览（5 分钟建立整体印象）](#0-项目全景速览)
- [1. 项目适配度评估](#1-项目适配度评估)
- [2. 前置知识点清单（必学 / 选学）](#2-前置知识点清单)
- [3. 项目模块化拆解](#3-项目模块化拆解)
- [4. 分阶段学习计划](#4-分阶段学习计划)
- [5. 项目逐模块深度学习引导](#5-项目逐模块深度学习引导)
- [6. 项目简历包装方案](#6-项目简历包装方案)
- [7. 附录](#7-附录)

---

## 0. 项目全景速览

### 0.1 一句话描述

**这个项目用 QEMU 虚拟出一块 PCI 显卡（GPU），自己写了它的 Linux 驱动、用户态运行时、显存分配器、GPU 汇编程序（算子），最后在上面完整跑通了 Qwen2-0.5B 大模型的推理（一个字一个字地生成文本）。**

### 0.2 完整软件栈（从上到下）

```text
┌─────────────────────────────────────────────────────┐
│  Python 工具层（host 端）                             │
│  tools/tokenize_qwen.py      文本 → token id         │
│  tools/detokenize_qwen.py    token id → 文本         │
│  tools/dump_weights.py       HuggingFace 权重 → .bin │
├─────────────────────────────────────────────────────┤
│  C 应用层（VM 内用户态）                              │
│  infer_tokens / test_transformer                    │
├─────────────────────────────────────────────────────┤
│  推理运行时（用户态 C 库）                             │
│  transformer.c   24 层解码器流程编排                  │
│  weight_loader.c 权重搬进 VRAM                        │
│  kv_cache.c      KV 缓存管理                         │
│  buddy.c + slab_cache.c   显存分配器                 │
├─────────────────────────────────────────────────────┤
│  libgpgpu 用户态驱动封装（open/mmap/ioctl/memcpy）     │
├─────────────────────────────────────────────────────┤
│  gpgpu_drv.ko  Linux PCI 字符设备驱动（内核态）        │
├─────────────────────────────────────────────────────┤
│  QEMU 虚拟 GPU 设备（不在本仓库，配套仓库）             │
│  gpgpu.c   PCI 设备/BAR/MMIO/中断                    │
│  gpgpu_core.c   RV32 SIMT 指令解释器（软件模拟 GPU 核）│
└─────────────────────────────────────────────────────┘
```

### 0.3 一次 kernel 执行的完整路径（本项目最重要的链路）

```text
用户态 gpuLaunchKernel()
    ↓ ioctl(GPGPU_IOC_DISPATCH)              【系统调用，用户态→内核态】
驱动 gpgpu_ioctl()                           [gpgpu_drv.c:114]
    ↓ iowrite32() 写 BAR0 上的 MMIO 寄存器      【内核态→设备】
QEMU gpgpu_ctrl_write() 收到 DISPATCH 写
    ↓ gpgpu_core_exec_kernel()               【设备侧解释执行 RV32 汇编】
QEMU timer 100ms 后触发 gpgpu_kernel_complete()
    ↓ pci_irq_assert()                       【设备→CPU 拉中断线】
驱动 irq_kernel_handler()                     [gpgpu_drv.c:76]
    ↓ kernel_done=1; wake_up_interruptible() 【唤醒等待队列】
ioctl 里 wait_event_interruptible() 醒来返回    【内核态→用户态】
用户态认为 kernel 执行完成，继续下一步
```

把这条链路吃透，你就理解了真实 GPU 驱动（AMD/NVIDIA/国产 GPU）工作的骨架：**用户态提交任务 → 驱动写设备寄存器 → 设备执行 → 中断通知完成 → 用户态继续**。

### 0.4 项目规格一览

| 项目 | 内容 |
|---|---|
| 设备类型 | QEMU 虚拟 PCI GPGPU（vendor 0x1234, device 0x1337） |
| BAR0 / BAR2 / BAR4 | 控制寄存器 1MB / VRAM（显存）/ Doorbell 64KB |
| 设备节点 | `/dev/gpgpu`（字符设备） |
| GPU 核心 | RV32 SIMT 解释器（软件模拟 32 线程 warp） |
| 手写算子 | gemm、rmsnorm、softmax、rope、silu 等 20+ 个汇编 kernel |
| 显存分配 | 用户态 buddy（伙伴系统）+ slab（小块缓存），管理 4GB VRAM |
| 跑通的模型 | Qwen2-0.5B-Instruct（24 层、896 隐藏维、2.35GiB FP32 权重） |
| 代码规模 | 驱动 320 行 + 用户态库约 5000 行 + 汇编约 1000 行 + Python 工具 |

### 0.5 概念科普：两个高频"关系题"

入门阶段面试最常考的两个概念关系，先在这里讲透，读到第 5 章时会有更深的体会。

#### 0.5.1 PCI 和 PCIe 是一回事吗？

**一句话：硬件上完全是两码事；从写驱动的角度看，基本一样。** 面试里"PCI 驱动""PCIe 驱动"经常混着说。

| | PCI（1992） | PCIe（2004 至今） |
|---|---|---|
| 传输方式 | **并行**总线，几十根信号线一起传 | **串行**，每方向一对差分线 |
| 拓扑 | 所有设备共享一条总线，谁通信谁占线 | 点到点，每个设备独享链路（x1/x4/x16 lane） |
| 现状 | 已淘汰 | 现在所有插槽都是它 |

PCIe 在设计时做了一个关键决定：**保留 PCI 的软件模型**。配置空间前 64 字节的寄存器布局（vendor ID / device ID / BAR / IRQ）、BAR 机制、枚举流程全部兼容——目的就是不让海量既有驱动推倒重写。所以 Linux 内核里只有一个 PCI 子系统：同一个 `struct pci_dev` / `struct pci_driver`，同一套 `pci_enable_device` / `pci_request_regions` / `pci_iomap` API。

**与本项目的联系**：[gpgpu_drv.c:168](gpgpu_driver/gpgpu_drv.c#L168) 的 `gpgpu_probe` 用的全是这套通用 API，代码里一个字都没提 PCI 还是 PCIe；而 QEMU `virt` 机器上那条总线其实是 **PCIe**（ECAM 方式访问配置空间）。也就是说：**你已经写过 PCIe 驱动了，只是没意识到**——这就是"软件兼容"的直接体现。

> **面试问答**
> 问：PCIe 设备能直接用为 PCI 写的驱动吗？
> 答：能。PCIe 设计时就保持了与 PCI 的软件兼容——配置空间模型、BAR、枚举流程全部一致，内核把两者统一在同一个 PCI 子系统里处理，驱动只面向 `struct pci_dev` 编程，不感知底层总线类型（所以 API 都叫 `pci_xxx`）。
> 追问：PCIe 在软件层面新增了什么？
> 答：MSI/MSI-X 消息式中断（取代共享的 INTx 引脚中断）、配置空间扩展到 4KB（capabilities 链：电源管理、AER 错误报告等）。基础模型一样，能力增量加入。

#### 0.5.2 GPU 和 GPGPU 是什么关系？

**一句话：GPU 是硬件；GPGPU = General-Purpose computing on GPU，指"把 GPU 当通用并行计算器用"这件事。同一个硅片，两种用法。**

- **GPU**（Graphics Processing Unit）诞生目的是画 3D 画面；**GPGPU** 是拿它算数学、跑 AI，不接任何显示器
- 为什么画图的芯片适合算数学：GPU 是"几千个小核心"的架构，天生适合 **SIMT**（一条指令、成千上万个线程各算各的数据）——矩阵乘、卷积、softmax 这类 AI 运算恰好全是"海量同构简单计算"
- 时间线：2000 年代早期有人用图形 API 偷偷做计算（"穷人的并行计算"）→ **2006 年 CUDA** 发布，GPGPU 正式成立 → 2010 年代深度学习爆发，数据中心 GPU 全是 GPGPU 用法（这也是 NVIDIA 市值神话的根源）

**与本项目的联系**：本项目就是 GPGPU 的最小模型——虚拟设备**没有任何显示输出**，只有三件事：VRAM（存权重和数据）、执行 RV32 SIMT kernel（ioctl 派发时写 grid/block/thread 参数，就是 CUDA 派发模型的寄存器版，见 [gpgpu_drv.c:114](gpgpu_driver/gpgpu_drv.c#L114)）、中断上报完成。`kernels/` 里全是 `matmul` / `softmax` / `rmsnorm` / `embedding_lookup` 等**大模型算子**，没有一个与图形相关；项目最终目标是跑 Qwen2-0.5B 推理。

> **面试问答**
> 问：GPU 和 GPGPU 的区别？
> 答：GPU 是硬件名称；GPGPU 指利用 GPU 做非图形类通用计算的使用方式。同一颗芯片，GPGPU 场景下不承担显示输出，通过 CUDA/OpenCL 这类编程模型把大规模同构计算以 SIMT 方式派发到数千核心并行执行。本人在项目中实现的虚拟 gpgpu 设备就是 GPGPU 的最小模型：无图形管线，只有显存、SIMT kernel 执行和中断上报。
> 追问：为什么 AI 用 GPU 而不是 CPU？
> 答：AI 的核心运算是矩阵乘等海量同构计算，GPU 数千核心的并行吞吐远高于 CPU 少数强核心；CPU 的强项（复杂控制流、分支预测、大缓存）在这类负载里用不上。这是"适合的架构做适合的事"——单线程跑复杂逻辑，CPU 反而完胜。

---

## 1. 项目适配度评估

### 1.1 总体结论

**匹配度：★★★★★（5/5）——对【嵌入式Linux驱动/BSP】岗位而言，这是一个罕见的"驱动 + 硬件交互 + 完整软件栈"三者齐全的项目，远超一般的点灯/I2C 练习项目。**

面试官看到这个项目会立刻产生兴趣，因为：

1. 它是**真正的 PCI 设备驱动**，而不是最基础的平台设备（platform）字符驱动——PCI 是 BSP 岗位高频考点。
2. 它打通了**完整调用链路**：用户态 → 系统调用 → 驱动 → MMIO 寄存器 → 中断 → 等待队列，每一环都有代码。
3. 项目有**大量真实踩坑记录**（开发日志里写了 bug 定位全过程），说明你不是"抄代码"，面试时可以讲出细节。

### 1.2 优势（面试时打什么牌）

| 优势 | 说明 | 对应简历关键词 |
|---|---|---|
| PCI 驱动全流程 | probe/remove、BAR 映射、中断注册、字符设备，教科书级骨架 | PCIe、BAR、MMIO、INTx 中断 |
| 用户态/内核态交互设计 | mmap 直通显存 + ioctl 提交任务，是 GPU/加速器驱动的事实标准做法 | mmap、ioctl、零拷贝、用户态驱动封装 |
| 中断 + 等待队列完成同步 | `wait_event_interruptible` + `wake_up`，是驱动开发最常见同步模式 | 等待队列、中断上下文、进程睡眠唤醒 |
| 内存分配器从零实现 | 在用户态复刻了 Linux 内核的 buddy + slab 设计 | 伙伴算法、slab、内存碎片、对齐 |
| 底层汇编能力 | 手写 RV32 汇编算子 + 自定义扩展指令（.word 手工编码） | RISC-V 汇编、指令编码、SIMT 模型 |
| 工程化调试方法 | bit-exact 对照测试、checksum 校验、逐层 dump | 测试方法学、调试方法论 |
| 完整系统思维 | 从设备模型到 AI 应用全栈打通，跨层理解能力强 | 软硬件协同、全栈 |

### 1.3 短板（必须诚实面对，面试官可能问）

| 短板 | 影响 | 应对策略 |
|---|---|---|
| QEMU 设备侧代码不在本仓库 | 面试官可能问"设备侧你理解多少" | 通读配套仓库设计文档（README 已详细描述），能把 BAR/MMIO/中断的设备侧逻辑讲清楚 |
| 驱动没有 DMA、没有 MSI-X、没有并发保护 | 面试官可能问"离真实 GPU 驱动差什么" | 主动说出这些简化点及改进方案（见第 6 章），反而加分 |
| 分配器是用户态实现（不是内核态） | 面试官可能问"为什么不放内核" | 能回答"toy 项目的取舍 + 真实现放哪的利弊"，说明你懂分层设计 |
| 不是原创项目（GitHub 下载） | 面试官可能问"你做了什么" | **不要撒谎**。本文第 6 章教你把"复现 + 深挖 + 小改进"包装成诚实的项目经历 |
| 性能是玩具级 | 面试官可能问"性能如何" | 能说清瓶颈在哪（每算子一次 ioctl、KV cache 走 D2H/H2D 往返）就是加分 |

### 1.4 简历可提炼的技术关键词

> 按岗位 JD 出现频率排序，这些词要能说出口并解释：

- **内核态**：PCI 设备驱动、字符设备、cdev、file_operations、probe/remove、BAR、MMIO、ioremap/`pci_iomap`、`io_remap_pfn_range`、中断（INTx）、`request_irq`、中断上下文、等待队列、`copy_from_user`、ioctl、设备模型（class/device）
- **用户态**：mmap、ioctl、零拷贝、用户态运行时库
- **内存管理**：buddy 伙伴算法、slab、内存对齐、碎片、round-up
- **体系结构**：RISC-V RV32、汇编、指令编码、CSR、SIMT、warp、lane、divergence
- **工程**：交叉编译、QEMU、bit-exact 测试、checksum、逐层 dump、gdb

### 1.5 面试官大概率深挖的内容（先剧透，第 6 章给答案）

1. `mmap()` 回调里 `vm_pgoff` 为什么能区分 BAR0/BAR2？（项目真实踩坑点）
2. 驱动等待 kernel 完成为什么用 `wait_event_interruptible` 而不是忙等（while 循环轮询）？
3. 中断处理函数里能做什么、不能做什么？为什么 handler 里只置标志位 + 唤醒？
4. buddy 分配器为什么要按 2 的幂向上取整？这导致项目里发生了什么问题？（2.35GB 权重装不下 → 拆 8 个 region 的故事）
5. `iowrite32` 和普通 C 指针赋值有什么区别？（内存屏障 / volatile / 总线序）
6. 如果两个进程同时打开 `/dev/gpgpu` 并发 launch，驱动会出什么问题？（没有并发保护）
7. 为什么 RISC-V 平台下不能用 MSI-X？（README 里的真实坑）
8. softmax 里空 lane 不初始化累加器为什么会让结果错？（开发日志里的真实 bug）

---

## 2. 前置知识点清单

> 原则：**只学本项目用得到的东西**，每个知识点都标注"它在本项目的哪里出现"，学完立刻能在代码里对上号。
> 你已了解字符设备基本框架，所以下面从"字符设备之上还缺什么"开始补。

### 2.1 【必学】清单（按学习顺序排列）

| # | 知识点 | 学习目标 | 学习重点 | 项目对应位置 | 建议时长 |
|---|---|---|---|---|---|
| 1 | **内核模块开发环境** | 会编译、加载、卸载一个 .ko，会用 dmesg 看日志 | Makefile 里 `obj-m`、`make -C $(KDIR) M=$(PWD)` 的含义；`insmod`/`rmmod`/`lsmod`；`printk` 日志级别；为什么模块必须和内核版本匹配 | [Makefile](gpgpu_driver/Makefile)、驱动里的 `printk`/`dev_info` | 2 天 |
| 2 | **字符设备驱动进阶** | 能把 cdev/file_operations/设备节点/class 讲成一个完整故事 | `cdev_init`+`cdev_add`、`alloc_chrdev_region`、`class_create`+`device_create` 各自职责；`container_of` 宏原理；`file->private_data` 的作用；`copy_from_user` 为什么必须有 | [gpgpu_drv.c:83-165](gpgpu_driver/gpgpu_drv.c#L83-L165) | 2 天 |
| 3 | **mmap 驱动实现** | 理解"用户态直接访问设备内存"的原理 | 虚拟地址 vs 物理地址；`vm_area_struct`、`vm_pgoff`（注意它是以 PAGE_SIZE 为单位的！）；`io_remap_pfn_range` 和 `remap_pfn_range` 的区别；为什么普通 malloc 的内存不能这样映射 | [gpgpu_drv.c:92-112](gpgpu_driver/gpgpu_drv.c#L92-L112) | 2 天 |
| 4 | **PCI 设备模型** | 能讲清 PCI 设备从发现到驱动的完整流程 | PCI 配置空间（vendor/device id 在哪）；BAR 是什么（设备向系统申请的内存窗口）；`pci_enable_device`/`pci_request_regions`/`pci_iomap` 各干什么；`probe` 被谁调用、什么时机调用；`pci_set_drvdata`/`pci_get_drvdata` | [gpgpu_drv.c:168-284](gpgpu_driver/gpgpu_drv.c#L168-L284) | 2 天 |
| 5 | **中断与等待队列** | 能讲清"进程睡眠等中断"这条同步路径 | 中断上半部/下半部概念；`request_irq` 参数含义（IRQF_SHARED！）；中断上下文里不能睡眠、不能用 copy_to/from_user；`wait_event_interruptible` 的宏展开原理（为什么外面要 while？本项目没写 while 会怎样）；`wake_up_interruptible` | [gpgpu_drv.c:76-81](gpgpu_driver/gpgpu_drv.c#L76-L81)、[gpgpu_drv.c:150-152](gpgpu_driver/gpgpu_drv.c#L150-L152) | 2 天 |
| 6 | **内核 I/O 访问** | 会用 ioread/iowrite 系列，知道为什么不能直接解引用指针 | `__iomem` 指针是什么；`iowrite32` vs `writel` vs 普通赋值；内存屏障的作用（写 DISPATCH 前要保证前面参数寄存器已到设备） | [gpgpu_drv.c:124-150](gpgpu_driver/gpgpu_drv.c#L124-L150) | 1 天 |
| 7 | **buddy/slab 算法概念** | 理解内核内存管理的两大机制 | buddy：按 2 的幂分块、分裂、合并（看本项目代码前先画图理解）；slab：为什么小对象要单独缓存、slab 页怎么切 slot | [buddy.c](gpgpu_driver/lib/src/buddy.c)、[slab_cache.c](gpgpu_driver/lib/src/slab_cache.c) | 2 天 |
| 8 | **RISC-V RV32 汇编基础** | 能逐行读懂 kernels/*.S | 寄存器 x0-x31 和 ABI 名（a0-a7 参数、t0-t6 临时、s0-s11 保存、fp/ra）；`lw/sw/flw/fsw` 读写；算术指令格式；`csrr` 读 CSR（mhartid）；伪指令（li、mv、beqz）会被展开成什么 | [kernels/gemm.S](gpgpu_driver/kernels/gemm.S) 全篇 | 3 天 |
| 9 | **GDB 调试（C 程序）** | 会用 gdb 断点、单步、看内存看变量 | break/next/step/print/x 命令；`layout src`；core dump 分析 | 调试 lib/test 下的测试程序 | 1 天 |

> 必学合计约 **17 天（2.5 周）**。这是本项目的地基，跳过任何一个都会导致后面读代码卡壳。

### 2.2 【选学】清单（有时间再补，不影响主线）

| # | 知识点 | 为什么可选 | 什么时候需要 |
|---|---|---|---|
| 10 | QEMU 设备模型（QOM、memory_region、PCI realize） | 设备侧代码不在本仓库，主线学习不需要 | 想端到端改设备行为时 |
| 11 | SIMT/GPU 编程模型（grid/block/warp 概念，可看 CUDA 入门） | 本项目 warp/lane 概念简单，README 解释够用 | 想深入理解 divergence/min-PC 调度时 |
| 12 | Transformer 推理基础（注意力、RoPE、KV cache、RMSNorm） | C 代码不要求懂数学原理就能读 | 想讲清楚"这个项目到底算出了什么"时（面试加分） |
| 13 | 内核并发（spinlock/mutex/RCU） | 本项目驱动恰好没做并发保护（这是一个可讲的改进点！） | 第 6 章"改进方向"讨论时 |
| 14 | DMA 与 IOMMU | 本项目用 mmap 直通替代了 DMA | 面试聊"真实 GPU 驱动怎么传数据"时 |

### 2.3 学习资源建议（按你的水平选）

- 内核模块/字符设备：宋宝华《Linux 设备驱动开发详解》前 8 章（有电子版）
- PCI：看 `Documentation/PCI/pci.rst`（内核源码树里自带）+ 本书 PCI 章节
- 中断/等待队列：《Linux 内核设计与实现》（LKD）第 7 章（中断）、第 4 章（进程调度与等待队列）
- RISC-V 汇编：《The RISC-V Reader》前 3 章（有中文版《RISC-V 开放架构设计之道》），够读本项目 .S 即可
- buddy/slab：LKD 第 11、12 章
- 动手环境：先在你自己电脑上装一个 x86 的 Linux 虚拟机（或直接用本机）跑内核模块实验，**不必一开始就上 RISC-V**

---

## 3. 项目模块化拆解

### 3.1 模块总览与依赖关系

```text
                    ┌──────────────┐
                    │  M8 测试体系  │ ← 横切所有模块（每个模块都有配套测试）
                    └──────────────┘
                          ▲
        ┌─────────────────┼─────────────────────┐
        │                 │                     │
   ┌────┴─────┐     ┌─────┴─────┐         ┌─────┴─────┐
   │ M7 推理层 │ ──▶ │ M6 KV缓存 │         │ M5 权重加载│
   │transformer│     │ kv_cache  │         │weight_loader│
   └────┬─────┘     └─────┬─────┘         └─────┬─────┘
        │                 │                     │
        └────────┬────────┴──────────┬──────────┘
                 ▼                   ▼
          ┌────────────┐      ┌────────────┐
          │ M4 GPU算子  │      │ M3 显存分配 │
          │ kernels/*.S │      │buddy+slab  │
          └─────┬──────┘      └─────┬──────┘
                │                   │
                └─────────┬─────────┘
                          ▼
                 ┌─────────────────┐
                 │ M2 用户态运行时  │ libgpgpu.c（open/mmap/ioctl/memcpy/launch）
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ M1 内核驱动      │ gpgpu_drv.c（PCI + 字符设备 + 中断）
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ M0 QEMU设备侧   │ （仓库外，配套 QEMU）
                 └─────────────────┘
```

### 3.2 各模块明细与学习顺序

| 顺序 | 模块 | 作用（大白话） | 代码目录 | 依赖 | 学习要点 |
|---|---|---|---|---|---|
| **M1** | Linux PCI 驱动 | 让内核认识这块"显卡"：申请资源、映射寄存器、暴露 `/dev/gpgpu`、把任务派发写进设备寄存器、等中断 | [gpgpu_drv.c](gpgpu_driver/gpgpu_drv.c)（320 行） | 无（代码里直接依赖 M0 设备的存在） | **你的核心模块**，简历主角，必须滚瓜烂熟 |
| **M2** | 用户态运行时 | 给上层 C 程序提供类 CUDA 的 API：`gpuInit/gpuMalloc/gpuMemcpy/gpuLaunchKernel` | [lib/libgpgpu.c](gpgpu_driver/lib/libgpgpu.c) + [libgpgpu.h](gpgpu_driver/lib/libgpgpu.h) | M1 | 理解"驱动暴露机制、用户态封装策略"的分工 |
| **M3** | VRAM 分配器 | 管理 4GB 显存：大块走 buddy、小块走 slab | [lib/src/buddy.c](gpgpu_driver/lib/src/buddy.c)、[lib/src/slab_cache.c](gpgpu_driver/lib/src/slab_cache.c)、[buddy.h](gpgpu_driver/lib/include/buddy.h)、[slab_cache.h](gpgpu_driver/lib/include/slab_cache.h) | M2 | 纯 C 算法，可以在**任何 Linux 机器上单独编译测试**（不依赖驱动！） |
| **M4** | GPU 汇编算子 | 20+ 个手写 RV32 汇编 kernel，是"在显卡上跑的代码" | [kernels/*.S](gpgpu_driver/kernels/)、[kernels/Makefile](gpgpu_driver/kernels/Makefile) | M2（参数约定） | 读汇编 + 理解 SIMT 编程模型 |
| **M5** | 权重加载器 | 把 2.35GiB 模型权重文件搬进 VRAM 并记住每层地址 | [lib/src/weight_loader.c](gpgpu_driver/lib/src/weight_loader.c)、[lib/include/weight_layout.h](gpgpu_driver/lib/include/weight_layout.h) | M2、M3 | 讲 buddy round-up 导致拆 8 块的真实故事 |
| **M6** | KV cache | 保存历史 token 的 K/V，让下一个 token 能"看到"之前的所有内容 | [lib/src/kv_cache.c](gpgpu_driver/lib/src/kv_cache.c)、[kv_cache.h](gpgpu_driver/lib/include/kv_cache.h) | M2、M3 | append/commit 两段式协议设计 |
| **M7** | Transformer 推理层 | 把 20 个算子按 Qwen2 的结构编排成完整解码流程 | [lib/src/transformer.c](gpgpu_driver/lib/src/transformer.c)（1278 行）、[transformer.h](gpgpu_driver/lib/include/transformer.h) | M3、M4、M5、M6 | 看"地址簿 + op 封装 + 大循环"三层结构，不需要懂 AI 数学 |
| **M8** | 测试与验证体系 | 每个算子有 host 参考实现对照测试；bit-exact 逐层 dump | [lib/test/*.c](gpgpu_driver/lib/test/)（30 个测试）、[tools/](gpgpu_driver/tools/)、[ref/](gpgpu_driver/ref/) | M2-M7 | 学"怎么证明驱动/算子是错的"，这是工程能力分水岭 |
| **M0** | QEMU 设备侧 | 虚拟显卡本身：BAR、MMIO、中断、RV32 解释器 | **不在本仓库**（配套仓库见 README 开头链接） | 无 | 只看 README 架构章节 + 配套仓库 `hw/gpgpu/` 浏览，理解设备视角 |

### 3.3 为什么按这个顺序学

1. **M1 驱动是简历核心**，且只依赖"内核机制"不依赖"上层业务逻辑"，最容易先啃下。
2. **M2 是 M1 的镜像**（驱动提供什么，runtime 就用什么），对照着读能加深对"驱动 API 设计"的理解。
3. **M3 纯用户态 C 算法**，可以立刻在本机编译跑测试——早期获得正反馈，且 buddy/slab 是面试高频题。
4. **M4 汇编**需要先有 M2 的参数约定背景（kernel 从 `a0` 指向的 VRAM 参数块取参）。
5. **M5/M6/M7 是应用逻辑**，面试权重低于驱动，但能讲出"我把 AI 跑在自研驱动上"是极大的差异化亮点。
6. **M8 贯穿全程**：学每个模块时都先跑对应测试，用测试反过来验证你对代码的理解。

---

## 4. 分阶段学习计划

> 总周期：**8 周**（按每天 2-3 小时业余时间设计；全职学习可压缩到 4-5 周）。
> 每阶段的产出物就是你的**面试弹药**：笔记（给面试官讲故事的素材）、代码注释（证明你真的读了）、自测实验（证明你真的动手了）。

### 4.0 环境准备（第 0 周，1-2 天）

**注意你本机的限制**：当前机器 3.8GB 内存、34GB 剩余磁盘。完整推理需要 QEMU 虚拟 4GB VRAM（QEMU 的 VRAM 就是 host 内存）+ 2.35GB 权重文件，**本机跑不动完整 Qwen 推理**。学习计划已按此设计：**大部分学习不依赖完整推理环境**。

| 环境 | 用途 | 本机可行性 |
|---|---|---|
| x86 Linux（本机）+ gcc | M3 分配器单独编译测试、读代码、gdb 调试 C 程序 | ✅ 现在就能开始 |
| RISC-V 交叉工具链（riscv64-linux-gnu-gcc、riscv64-unknown-elf-gcc） | 编译驱动 .ko 和 kernel 汇编 | ✅ 需要安装（apt install gcc-riscv64-linux-gnu gcc-riscv64-unknown-elf，或用项目 kernel-headers 里要求的工具链） |
| QEMU（带 gpgpu 设备）+ RISC-V Linux VM | 跑驱动、跑推理 | ⚠️ 本机内存不够，建议：a) 借一台 ≥16GB 的机器/云服务器；b) 只装 QEMU 学习 M0 概念；c) 后期有条件再搭 |
| Qwen 权重文件 2.35GiB | M5/M7 端到端 | ⚠️ 磁盘够，但配合推理需要内存 |

**第 0 周任务**：
1. 安装交叉工具链，把 [kernels/Makefile](gpgpu_driver/kernels/Makefile) 的 `CROSS := riscv64-unknown-elf` 改成你装到的前缀（或安装对应工具链），跑 `make list` 确认工具链可用。
2. 用 `git log --oneline` 通读一遍 24 条提交记录，按提交顺序把代码当"项目开发史"看一遍（每条提交干了什么，正好对应模块拆解）。
3. 通读 README 一遍（30 分钟），重点看"设计笔记"和"调试经验"两节。

### 阶段 1：前置知识补齐（第 1-2.5 周）

**目标**：掌握 2.1 节全部必学知识点，每个知识点都能在本项目代码里指出对应位置。

**任务**（按 2.1 表格顺序）：
- 知识点 1-2：在自己电脑上写一个最小字符设备驱动（参考宋宝华书），`insmod` 后 `echo/cat` 能读写设备，`dmesg` 看 printk。
- 知识点 3：给上面的驱动加 `mmap`，用户态 mmap 后写内存、驱动侧看数据。
- 知识点 4：`lspci -v` 看你自己电脑的 PCI 设备列表，找一个设备对照理解 BAR 概念。
- 知识点 5：给驱动加一个 ioctl，ioctl 里 `wait_event_interruptible` 睡眠，另起一个内核线程（或定时器）唤醒它。
- 知识点 7：手写一个 100 行的 buddy 分配器（先不看本项目的！写完再对照 [buddy.c](gpgpu_driver/lib/src/buddy.c) 找差异）。
- 知识点 8：用 RISC-V 汇编写一个"数组求和"小程序，在 QEMU user 模式（`qemu-riscv32`）或 rv32 模拟器上跑通。

**产出物**：
- `my_driver/`：自己的字符设备 + mmap + ioctl + 等待队列练习代码（**这个本身就是简历素材的一部分**）
- 笔记：`前置知识笔记.md`（每个知识点一页：概念图 + 本项目对应代码行号）
- 手写 buddy 分配器 + 单元测试

**自测实验**：能把下面这些问题不看答案答出来：
- probe 为什么不是 main 函数？谁调用它？
- `vm_pgoff == 1` 时映射的是哪个 BAR？为什么乘 PAGE_SIZE？

### 阶段 2：驱动模块深读 M1 + M2（第 2.5-3.5 周）

**目标**：把 [gpgpu_drv.c](gpgpu_driver/gpgpu_drv.c) 每一行吃透，能默写 probe 流程。

**任务**：
1. 按第 5 章 M1 导读逐段精读，边读边在代码里写中文注释（建议复制一份到 `学习笔记/` 下注释，**别直接改原仓库**）。
2. 对照 M2 [libgpgpu.c](gpgpu_driver/lib/libgpgpu.c) 读，画一张"驱动提供的能力 ↔ 用户态如何使用"对照表。
3. 找出驱动里的 3 个 bug/隐患（提示：ioctl 被信号打断的处理、wait_event 外面没 while、IRQF_SHARED 但 handler 不判断是不是自己的中断）。这 3 个点面试会问到！
4. 若有 QEMU 环境：按 README 编译驱动、insmod、跑 [test_gpgpu.c](gpgpu_driver/test/test_gpgpu.c)，用 `dmesg` 观察 probe 日志、`lspci` 看设备。

**产出物**：
- `M1驱动注释版.c`（全行注释）+ `M1知识点清单.md`（每个内核 API 的用途/参数/返回值）
- `驱动-用户态接口对照表.md`
- `驱动3个隐患分析.md`（每个：现象、原因、修复代码）

**自测实验**：写一个用户态程序并发 launch 两个 kernel（pthread），观察 `wait_event` 的行为；再试试 ioctl 睡眠时 Ctrl+C 会发生什么。

### 阶段 3：分配器 + 汇编算子 M3 + M4（第 3.5-5 周）

**目标**：buddy/slab 代码逐行理解；能读懂每个 kernel 汇编；理解 SIMT 编程模型。

**任务**：
1. 按第 5 章 M3 导读精读，重点搞懂 `page_order[]` 数组如何身兼两职（buddy 块阶数 / slab 页 marker）。
2. 本机编译运行 [test_buddy.c](gpgpu_driver/lib/test/test_buddy.c) 和 [test_slab.c](gpgpu_driver/lib/test/test_slab.c)（它们依赖 gpuInit 吗？读了就知道——test_buddy 直接测试 buddy 结构，可本机跑）。
3. 按第 5 章 M4 导读读汇编，顺序：gemm → rmsnorm → softmax_decode → rope → argmax → 其余。
4. 用 `make NAME.dump` 看反汇编，验证伪指令展开（如 `li` 展开成什么）。
5. 手写一个自己的 kernel（例如 `vec_scale.S`：数组乘标量），照着 kernels/Makefile 编译出 .bin/.dump。

**产出物**：
- `M3分配器注释版.c` + buddy/slab 各画一张图（分裂/合并过程）
- `M4汇编注释版/`（每个 kernel 逐行注释）
- 自己的第一个 kernel：`vec_scale.S`（这就是简历里"扩展算子"的素材）

**自测实验**：改 slab 的 object_size 为 100 字节，观察发生了什么（对齐问题！）；给 buddy 写一个"打印当前所有 free block"的调试函数。

### 阶段 4：推理软件栈 M5 + M6 + M7（第 5-7 周）

**目标**：理解 2.35GB 权重如何进显存、KV cache 协议、transformer 编排结构。

**任务**：
1. 精读 [weight_layout.h](gpgpu_driver/lib/include/weight_layout.h)：搞懂每个 offset 怎么算出来的（用计算器验算一遍 `WL_OFF_Q_PROJ_W = 3584` 从哪来）。
2. 精读 [weight_loader.c](gpgpu_driver/lib/src/weight_loader.c)：重点理解 8 个 region 的划分逻辑和"先 stat 校验再分配"的顺序设计。
3. 精读 [kv_cache.c](gpgpu_driver/lib/src/kv_cache.c)：搞懂 append/commit 两段式协议解决了什么问题（半途崩溃怎么办）。
4. 通读 [transformer.c](gpgpu_driver/lib/src/transformer.c)：先看结构不看细节——`transformer_init`（资源准备）→ `op_*`（算子封装）→ `transformer_decoder_layer`（单层流程）→ `transformer_step`（完整一步）。画出 scratch buffer 复用图。
5. 有环境的话：跑 `test_transformer`，看到 `prefill last (expected 3): 3 PASS` 即端到端打通。
6. 读 3 篇开发日志（docs/5.9、5.12、5.23），总结里面的 bug 故事（面试讲故事用）。

**产出物**：
- `权重布局验算.xlsx/md`（每个 tensor 的 offset 计算过程）
- `KV缓存协议笔记.md`（append/commit 状态机图）
- `transformer执行流程图.md`（一个 token 从 embedding 到 argmax 经过的 24 层 × 20+ 次 kernel launch 的完整流水）
- `踩坑故事集.md`（从开发日志提炼 5 个 bug 故事：现象→定位→修复→教训）

**自测实验**：把 KV cache 的 `max_seq` 从 128 改成 4，观察推理在第 5 个 token 时发生什么。

### 阶段 5：测试与调试体系 M8（第 7 周）

**目标**：掌握 bit-exact 对照方法，能讲清"怎么证明我的驱动/算子是错的"。

**任务**：
1. 读 [test_gemm.c](gpgpu_driver/lib/test/test_gemm.c) 等 2-3 个测试：搞懂 host reference 怎么写、误差怎么算。
2. 读 [tools/check_layer.py](gpgpu_driver/tools/check_layer.py)：搞懂逐层 dump 对照流程。
3. 读 docs/5.23 的 softmax bug 修复全过程（第 5 章 M4 有导读）。
4. 跑一次 DUMP_INTERMEDIATES 编译，看 dump/ 目录结构（有环境的话）。

**产出物**：`测试方法学笔记.md`（对照测试三要素：独立 reference、边界采样、误差阈值）

### 阶段 6：简历包装与模拟面试（第 8 周）

**目标**：完成简历、口述脚本、面试问答库。

**任务**：执行第 6 章的全部清单（简历两个版本、2 分钟口述脚本、20 个追问答案自测）。
**产出物**：`简历-驱动方向.md`、`简历-BSP方向.md`、`面试问答库.md`。

---

## 5. 项目逐模块深度学习引导

> 讲解约定：
> - 每个模块按「先讲人话 → 代码导读（跟着行号走）→ 关键内核概念解释 → 动手实验 → 面试思考题」展开。
> - 代码引用都可点击。建议把代码复制到自己的笔记目录里写注释。
> - 每个模块标注难度：⭐（轻松）~ ⭐⭐⭐⭐⭐（硬骨头）。

---

### 5.1 M1：Linux PCI 驱动（⭐⭐⭐⭐ 核心模块，学 3-4 天）

#### 5.1.1 先讲人话

你的电脑里插的显卡、网卡、NVMe 硬盘都是 **PCI 设备**。每个 PCI 设备出厂时自带一个"身份证"（vendor ID + device ID，存在 PCI 配置空间里）。Linux 启动时逐个扫描 PCI 总线上的设备，把身份证和所有已注册驱动的"通缉令"（`pci_device_id` 表）比对，比中了就调用那个驱动的 `probe()` 函数，说："这个设备归你管了，去初始化吧！"

本项目的驱动就是这张通缉令 + 接管流程：接管后它要干三件事——**把设备的寄存器窗口映射到自己能访问的地址**（不然碰不到硬件）、**注册一个字符设备节点**（让用户态程序能 open 它）、**注册中断**（设备干完活要能通知到驱动）。

#### 5.1.2 代码导读（按 probe 执行顺序）

**① 通缉令：[gpgpu_drv.c:69-74](gpgpu_driver/gpgpu_drv.c#L69-L74)**

```c
static struct pci_device_id gpgpu_pci_ids[] = {
    {PCI_DEVICE(GPGPU_VENDOR_ID, GPGPU_DEVICE_ID)},   // 0x1234, 0x1337
    {0, }};   // 空条目 = 表结束标志
MODULE_DEVICE_TABLE(pci, gpgpu_pci_ids);
```

`MODULE_DEVICE_TABLE` 宏让这条记录进到模块文件的特定段里，这样即使模块没加载，`modprobe` 也能按 ID 找到该加载哪个模块。QEMU 虚拟的显卡就是把自己伪装成 vendor=0x1234、device=0x1337 的设备，好让这个驱动认领。

**② 结构体"设备上下文"：[gpgpu_drv.c:48-58](gpgpu_driver/gpgpu_drv.c#L48-L58)**

```c
struct gpgpu_dev {
    struct pci_dev *pdev;          // 内核的 PCI 设备对象
    void __iomem   *ctrl_base;     // BAR0 映射后的虚拟地址（控制寄存器）
    void __iomem   *vram_base;     // BAR2 映射后的虚拟地址（显存）
    void __iomem   *doorbell_base; // BAR4 映射后的虚拟地址
    dev_t           devno;         // 主/次设备号
    struct cdev     cdev;          // 字符设备对象
    struct class *class;
    wait_queue_head_t wait_queue;  // 等待队列头（同步用）
    int               kernel_done; // 完成标志
};
```

**概念解释 —— BAR**：BAR（Base Address Register，基地址寄存器）是 PCI 设备向系统"要地址"的窗口。设备说"我需要一块 1MB 的内存窗口放控制寄存器"，系统启动时给它分配一段物理地址，写回 BAR。CPU 读写这段地址，PCI 控制器就会路由到这台设备上。本项目有 3 个 BAR：BAR0（控制寄存器，驱动往这里写命令、设备状态从这里读）、BAR2（显存，GPU 的"内存"）、BAR4（doorbell，门铃，以后优化提交用，当前未用）。

**概念解释 —— `__iomem`**：这是内核给"设备内存映射后得到的虚拟地址"打的标记。它不是普通 RAM 地址，**不能直接解引用**（`*p = 1` 是错的），必须用 `ioread32/iowrite32` 等函数访问，因为：a) 这种内存不走 CPU 缓存，直接访问会看到旧数据；b) 编译器可能把普通指针的读写优化重排，而设备寄存器读写顺序是有硬件含义的。

**③ probe 的错误处理骨架：[gpgpu_drv.c:168-177](gpgpu_driver/gpgpu_drv.c#L168-L177)**

```c
static int gpgpu_probe(struct pci_dev *pdev, const struct pci_device_id *id) {
    int ret = -ENOMEM;
    struct gpgpu_dev *gpu_dev = kmalloc(sizeof(struct gpgpu_dev), GFP_KERNEL);
    if (!gpu_dev) {
        printk("molloc memory failed\n");
        goto err_free;
    }
    gpu_dev->pdev = pdev;
    ...
```

先 `kmalloc` 一个私有结构，后面所有资源都挂在这个结构上（内核驱动最常见的"私有数据"模式）。注意 probe 全程用 **goto 错误链**：内核态不能像用户态那样 exit 了事，每一步申请的资源都要在失败时逆序释放，否则就是资源泄漏（模块卸载不掉、下次 probe 挂）。读 [gpgpu_drv.c:259-283](gpgpu_driver/gpgpu_drv.c#L259-L283) 的错误标签，体会这个"申请一层包一层"的洋葱结构。

**④ 启用设备 + 认领资源：[gpgpu_drv.c:186-196](gpgpu_driver/gpgpu_drv.c#L186-L196)**

```c
ret = pci_enable_device(pdev);        // 打开设备电源/总线功能，分配 I/O 资源
...
ret = pci_request_regions(pdev, "gpgpu");  // 声明"这些 BAR 我要用"，防止别的驱动抢
...
gpu_dev->ctrl_base = pci_iomap(pdev, 0, 0);   // 把 BAR0 的物理地址映射成虚拟地址
```

**概念解释**：`pci_iomap` 内部做的事情就是你学过的 `ioremap`（物理地址 → 内核虚拟地址），只是它帮你从 BAR 里读出物理地址再映射，并额外做了内存属性设置（可写回缓存等）。

**⑤ 读设备信息：[gpgpu_drv.c:212-218](gpgpu_driver/gpgpu_drv.c#L212-L218)**

```c
DEV_ID = ioread32(gpu_dev->ctrl_base + 0x00);
u64 vram_size = ((u64)ioread32(gpu_dev->ctrl_base + GPGPU_REG_VRAM_SIZE_HI) << 32)
              | ioread32(gpu_dev->ctrl_base + GPGPU_REG_VRAM_SIZE_LO);
```

驱动**第一次和硬件对话**：读 BAR0 偏移 0 处的设备 ID，读 VRAM 大小寄存器。VRAM 大小是 64 位的，但 MMIO 寄存器只有 32 位宽，所以分高/低两个寄存器（`_HI`/`_LO` 后缀），驱动拼起来。**这种"硬件寄存器 32 位、软件概念 64 位"的拆分在真实硬件里非常常见**（比如 64 位 DMA 地址、64 位计数器）。

**⑥ 注册字符设备：[gpgpu_drv.c:221-241](gpgpu_driver/gpgpu_drv.c#L221-L241)**

```c
ret = alloc_chrdev_region(&gpu_dev->devno, 0, 1, "gpgpu");  // 申请设备号
cdev_init(&gpu_dev->cdev, &gpgpu_fops);                      // 绑定操作函数表
ret = cdev_add(&gpu_dev->cdev, gpu_dev->devno, 1);           // 把 cdev 挂进内核
gpu_dev->class = class_create("gpgpu");                      // 创建 sysfs 类
dev = device_create(gpu_dev->class, NULL, gpu_dev->devno, NULL, "gpgpu"); // 生成 /dev/gpgpu
```

这是你学过的字符设备四件套，但注意顺序：**设备号 → cdev_init → cdev_add → class → device_create**。`device_create` 会通知 udev 在 `/dev` 下自动创建设备节点，用户态才能 `open("/dev/gpgpu")`。

**⑦ 注册中断：[gpgpu_drv.c:244-256](gpgpu_driver/gpgpu_drv.c#L244-L256)**

```c
pci_set_master(pdev);   // 允许设备发起总线事务（DMA 等需要）
ret = pci_alloc_irq_vectors(gpu_dev->pdev, 1, 1, PCI_IRQ_LEGACY);
ret = request_irq(pci_irq_vector(gpu_dev->pdev, 0), irq_kernel_handler,
                  IRQF_SHARED, "gpgpu-kernel", gpu_dev);
iowrite32(GPGPU_IRQ_KERNEL_DONE | GPGPU_IRQ_DMA_DONE | GPGPU_IRQ_ERROR,
          gpu_dev->ctrl_base + GPGPU_REG_IRQ_ENABLE);
```

**概念解释 —— INTx vs MSI/MSI-X**：传统中断 INTx 是设备拉一根共享的物理中断线；MSI/MSI-X 是设备直接往 CPU 写消息触发中断（更快、每条消息带向量号）。本项目用 `PCI_IRQ_LEGACY` 是因为 **RISC-V virt 平台的中断控制器（PLIC）不支持 MSI-X**——这是 README"调试经验"里记录的真实坑。面试官如果问"为什么不用 MSI-X"，这就是答案。

`IRQF_SHARED` 表示这根中断线可能被多个设备共享，handler 里本该检查"中断是不是我的设备发的"（读设备的中断状态寄存器），**但这个驱动没有检查**——这是可以主动讲的改进点。

#### 5.1.3 四个文件操作函数的深读

**open：[gpgpu_drv.c:83-88](gpgpu_driver/gpgpu_drv.c#L83-L88)** —— 用 `container_of` 从 cdev 找回大结构体，存进 `file->private_data`。**概念解释 —— container_of**：cdev 是 `struct gpgpu_dev` 里的一个成员，内核只给你成员指针，通过"成员地址 - 成员在结构体里的偏移"算出结构体首地址。这是内核里最重要的宏之一，面试必问，请务必手写一遍它的实现。

**mmap：[gpgpu_drv.c:92-112](gpgpu_driver/gpgpu_drv.c#L92-L112)**

```c
switch (vma->vm_pgoff) {
case 0:  pfn = pci_resource_start(gpgpu_dev->pdev, 0) >> PAGE_SHIFT; break;  // BAR0
case 1:  pfn = pci_resource_start(gpgpu_dev->pdev, 2) >> PAGE_SHIFT; break;  // BAR2
default: return -EINVAL;
}
ret = io_remap_pfn_range(vma, vma->vm_start, pfn, vma->vm_end - vma->vm_start,
                         vma->vm_page_prot);
```

用户态调用 `mmap(NULL, size, ..., fd, offset)` 时，offset 会按 4096 对齐后放进 `vma->vm_pgoff`。**关键设计**：驱动用这个 offset 当"选择器"——0 表示映射 BAR0（控制寄存器），1 表示映射 BAR2（显存）。所以用户态映射显存必须写 `mmap(..., fd, 1 * PAGE_SIZE)`。

**这是本项目第一个大坑**（README 里写了）：如果用户态 offset 写 0，会映射到 BAR0，程序以为自己往显存写数据，实际在改控制寄存器！这种"用一个整数参数复用出多个语义"的接口设计，面试时讲出来非常加分。

**概念解释 —— `io_remap_pfn_range`**：把**物理页框号（pfn）**映射到用户进程的虚拟地址空间，建立页表项。和 `remap_pfn_range` 的区别是它给映射打了"设备内存"标记（不可投机读、不走部分缓存），ARM64/RISC-V 上还会配相应的内存属性。为什么能直接映射物理地址？因为 BAR 地址是系统启动时统一分配的固定物理地址，不会像普通内存页那样被换出。

**ioctl：[gpgpu_drv.c:114-159](gpgpu_driver/gpgpu_drv.c#L114-L159)** —— 驱动最核心的操作：

```c
case GPGPU_IOC_DISPATCH:
    if (copy_from_user(&args, (void __user *)arg, sizeof(args)))
        return -EFAULT;
    gpgpu_dev->kernel_done = 0;
    iowrite32((uint32_t)(args.kernel_addr & 0xFFFFFFFF),
              gpgpu_dev->ctrl_base + GPGPU_REG_KERNEL_ADDR_LO);
    ... /* 依次写 KERNEL_ADDR、KERNEL_ARGS、GRID_DIM、BLOCK_DIM、SHARED_MEM_SIZE */
    iowrite32(1, gpgpu_dev->ctrl_base + GPGPU_REG_DISPATCH);  // 最后写 DISPATCH = 启动！
    wait_event_interruptible(gpgpu_dev->wait_queue, gpgpu_dev->kernel_done);
```

三步曲：
1. `copy_from_user` 把用户态参数安全拷进内核（用户指针不能直接解引用——它可能无效、可能指向被换出的页，直接碰会 oops）。
2. 把参数逐个 `iowrite32` 进 BAR0 的控制寄存器。**写 DISPATCH 寄存器 = 给设备下"开工"命令**。注意顺序：DISPATCH 一定最后写，且 `iowrite32` 内含内存屏障，保证前面所有寄存器写已到达设备后才真正触发执行——这就是"寄存器编程的时序协议"。
3. `wait_event_interruptible` 让当前进程**睡眠**，等 `kernel_done` 变 1。**概念解释**：睡眠 = 进程被移出运行队列，CPU 让给别人；等中断来了唤醒它。为什么不用 while 轮询？轮询会 100% 占用 CPU 傻等 100ms，睡眠则 CPU 完全空闲。`wait_event_interruptible` 内部是"检查条件 → 不满足则加入等待队列睡眠 → 被唤醒后再检查"的循环，所以**规范的写法外面不用再包 while**（但本项目裸用没有处理"被信号唤醒"的返回路径，见思考题）。

**中断 handler：[gpgpu_drv.c:76-81](gpgpu_driver/gpgpu_drv.c#L76-L81)**

```c
static irqreturn_t irq_kernel_handler(int irq, void *data) {
    struct gpgpu_dev *gpu_dev = data;
    gpu_dev->kernel_done = 1;
    wake_up_interruptible(&gpu_dev->wait_queue);
    return IRQ_HANDLED;
}
```

QEMU 那边执行完 kernel 后拉中断线，内核调这个函数。它只做两件事：置标志、唤醒等待队列。**概念解释 —— 中断上下文**：handler 运行在"中断上下文"里，不是任何进程的上下文，**不能睡眠**（没有可睡的进程）、**不能调 copy_to/from_user**（当前进程和被中断的进程无关）。所以这里只置个 int 标志、唤醒队列（唤醒本身不睡眠），真实工作让睡着的进程醒来继续做。这是 Linux 中断处理的黄金法则，面试 100% 会问。

#### 5.1.4 动手实验

1. **本机即可**：把 [gpgpu_drv.c](gpgpu_driver/gpgpu_drv.c) 复制到笔记目录，逐行写注释（每个 API 一句话：干什么、参数、返回值）。
2. 有 x86 Linux 环境的话，写一个"假 gpgpu"平台设备版驱动（不用 PCI，用 module_init 直接注册），把 ioctl+waitqueue+中断（用内核定时器模拟中断）路径跑一遍。
3. 有 RISC-V 交叉工具链：`make driver` 编译出 .ko（注意 Makefile 里 `KDIR` 指向仓库自带 kernel-headers，`CROSS_COMPILE` 需要你装的工具链）。
4. 有 QEMU 环境：insmod 后 `dmesg | grep gpgpu`，依次验证：probe 日志 → 设备 ID → VRAM 大小 → `/dev/gpgpu` 存在 → 跑 test_gpgpu 看到 mmap 读回 0x12345678。

#### 5.1.5 面试思考题（先自己答，答案见第 6 章）

1. `ioctl` 里 `wait_event_interruptible` 睡眠期间用户 Ctrl+C，会发生什么？`kernel_done` 还会被处理吗？
2. 两个进程同时打开设备并发 ioctl dispatch，会有什么问题？怎么修？
3. 中断 handler 为什么不直接 `copy_to_user` 把结果给用户态？
4. `mmap` 里为什么用 `vm_pgoff` 而不是传进来的 `vma->vm_start` 判断映射哪个 BAR？
5. probe 失败路径为什么是 goto 链而不是直接 return？找一个 goto 链里漏释放的资源（提示：看 `err_request_irq_kernel` 标签）。

---

### 5.2 M2：用户态运行时 libgpgpu（⭐⭐ 学 1-2 天）

#### 5.2.1 先讲人话

驱动给了用户态 4 个能力（open、mmap、ioctl、close），libgpgpu 把这 4 个能力包装成对上层友好的 API：初始化（`gpuInit`）、分配显存（`gpuMalloc`）、拷数据（`gpuMemcpy`）、启动 kernel（`gpuLaunchKernel`）。这就是"**机制在内核、策略在用户态**"的分层思想：内核只提供最原始的机制，复杂的显存管理、算子调度全在用户态做——真实 GPU 驱动也是这个路线（内核驱动只做最小工作，重活都在用户态驱动如 Mesa/CUDA 运行时里）。

#### 5.2.2 代码导读

**gpuInit：[libgpgpu.c:61-121](gpgpu_driver/lib/libgpgpu.c#L61-L121)** —— 三件事：

```c
ctx->fd = open(dev_path, O_RDWR);
ctx->vram_mmap = mmap(NULL, vram_size, PROT_READ | PROT_WRITE, MAP_SHARED,
                      ctx->fd, 1 * PAGE_SIZE);   // offset=4096 → vm_pgoff=1 → BAR2
if (buddy_init(&ctx->buddy, (uint64_t)ctx->vram_mmap, vram_size) < 0) ...
// 初始化 6 个 slab cache（32/64/128/256/512/2048 字节）
```

注意 mmap 的 offset 是 `1 * PAGE_SIZE`，对应驱动里的 `case 1`（BAR2 显存）。**mmap 之后，显存就像一块 malloc 出来的普通内存一样可以直接读写了**——这就是后面 `gpuMemcpy` 只是 memcpy 的原因。

**gpuMalloc 的双路分发：[libgpgpu.c:132-153](gpgpu_driver/lib/libgpgpu.c#L132-L153)**

```c
uint64_t gpuMalloc(gpgpu_ctx *ctx, size_t size) {
    struct slab_cache *sc = route_slab(ctx, size, &marker);
    if (sc != NULL) {          // ≤ 2048 字节 → slab
        uint64_t offset = slab_alloc(sc);
        return offset;
    }
    return buddy_alloc(&ctx->buddy, size);   // ≥ 4KB → buddy
}
```

小对象走 slab（避免为一个 32 字节的 kernel 参数浪费整页），大对象走 buddy。返回值是**显存偏移量**（VRAM 内的相对地址），不是宿主指针——所有上层代码都用"offset"这个概念在设备地址空间里思考，这是 GPU 编程的基本素养。

**gpuFree 的 marker 机制：[libgpgpu.c:156-165](gpgpu_driver/lib/libgpgpu.c#L156-L165)**

```c
int gpuFree(gpgpu_ctx *ctx, uint64_t offset) {
    uint32_t page_idx = offset / PAGE_SIZE;
    uint8_t  marker   = ctx->buddy.page_order[page_idx];   // 关键！
    struct slab_cache *sc = get_slab(ctx, marker);
    if (sc != NULL) return slab_free(sc, offset);
    return buddy_free(&ctx->buddy, offset);
}
```

`gpuFree` 只收到一个 offset，怎么知道它属于哪个 slab 还是 buddy？答案在 `page_order[]` 数组：slab 页的 `page_order` 存的是 0xF0~0xF5 的"标记值"，buddy 页存的是阶数（0~20）。**一个数组身兼两职**，用高位的特殊值区分类型——这是本项目最精巧的设计之一，面试时讲清楚它很加分。

**gpuLaunchKernel：[libgpgpu.c:182-197](gpgpu_driver/lib/libgpgpu.c#L182-L197)** —— 填好 `gpgpu_dispatch_args` 结构体，一个 ioctl 提交。注意结构体字段顺序和驱动里的定义**必须完全一致**（[libgpgpu.h:25-31](gpgpu_driver/lib/libgpgpu.h#L25-L31) 与 [gpgpu_drv.c:60-66](gpgpu_driver/gpgpu_drv.c#L60-L66) 对照）——ioctl 的 `_IOW` 宏把结构体大小也编码进了命令号，两边不一致会直接报错（内核会检查 `_IOC_SIZE`）。

#### 5.2.3 动手实验

1. 画一张"M1 驱动能力 ↔ M2 用户态调用"对照表。
2. 本机编译 [test_libgpgpu.c](gpgpu_driver/lib/test/test_libgpgpu.c)（没有设备会 fail，但能看代码流程）；有环境就跑它。

#### 5.2.4 面试思考题

1. `mmap` 映射 4GB 但 QEMU 的 BAR2 只有 4GB，如果设备侧 VRAM 实际更小（比如 64MB）会发生什么？（SIGBUS 的知识点）
2. `gpuMemcpy` 用 memcpy 直接操作 mmap 内存，和真实 GPU 的 DMA 拷贝相比差在哪？
3. 为什么要用户态做显存分配，而不是像 kmalloc 一样在驱动里分配？

---

### 5.3 M3：VRAM 分配器 buddy + slab（⭐⭐⭐⭐ 学 2-3 天）

#### 5.3.1 先讲人话

显存 4GB 是一整块连续空间。如果每次要多大就切多大、用完直接丢，很快会碎成"总空间够但找不到一块连续大空间"的局面（**外部碎片**）。Linux 内核用 buddy 算法管物理内存，用 slab 管内核对象，本项目在用户态复刻了这两个设计来管显存。

**buddy 思想**：所有空闲块按"2 的幂大小"分档管理（4KB、8KB、16KB……2GB）。要 100KB？向上取整到 128KB 档。该档没有空闲块？就把更大的块一分为二、二分为四，直到得到合适大小；释放时看它的"孪生兄弟"（buddy，即同一大块切出来的另一半）是否空闲，是就合并回上一档。**优点**：合并简单、大块不碎；**缺点**：向上取整浪费空间（内部碎片）。

**slab 思想**：小对象（32~2048 字节）单独处理。从 buddy 拿一页（4KB），切成固定大小的"槽位"（slot）串成链表，分配/释放都是 O(1) 的链表头插/头摘。**优点**：快、不浪费页；**缺点**：要维护"这页属于哪个 slab"的元信息。

#### 5.3.2 代码导读

**数据结构：[buddy.h:5-17](gpgpu_driver/lib/include/buddy.h#L5-L17)**

```c
#define BUDDY_MAX_ORDER 21        // 阶数 0~20，最大块 2^20 × 4KB = 4GB
#define BUDDY_TOTAL_PAGES 1048576 // 4GB / 4KB = 100 万页
struct buddy_node { struct buddy_node *prev, *next; };
struct buddy_allocator {
    struct buddy_node *free_list[BUDDY_MAX_ORDER];  // 21 条空闲链表，每条对应一个阶数
    uint8_t            page_order[BUDDY_TOTAL_PAGES]; // 每页记录：所属块的阶数 或 slab 标记
    uint64_t           vram_base;
};
```

两个精妙点：
1. `free_list[i]` 是 21 条双向链表，每格一个阶数。
2. `page_order[]` 是一张 1MB 的"户籍表"，每页 1 字节：0~20 = 该页属于某个阶数的 buddy 块（只记录块首地址对应的页）；0xF0~0xF5 = 该页是 slab 页。**内核的做法是用 struct page 数组，本项目用 1 字节数组，本质一样**。

**初始化（把 4GB 切成 2 的幂块挂链表）：[buddy.c:15-42](gpgpu_driver/lib/src/buddy.c#L15-L42)**

注意它不是简单地把整个 4GB 挂到 order 20：它从大到小尝试，`remaining >= block_size` 就切一块挂上——这是为了支持**非 2 幂大小**的显存（比如 3.5GB 也能初始化成 2GB+1GB+512MB 的组合）。

**分配（找 → 分裂 → 标记）：[buddy.c:50-98](gpgpu_driver/lib/src/buddy.c#L50-L98)**

```c
for (order = 0; order < BUDDY_MAX_ORDER; order++)   // 1. 算需要的最小阶数
    if ((1ULL << order) * PAGE_SIZE >= alloc_size) break;
for (i = order; i < BUDDY_MAX_ORDER; i++)           // 2. 从这一档往上找非空链表
    if (alloc->free_list[i]) break;
struct buddy_node *left = alloc->free_list[i];      // 3. 摘出这块
while (i > order) {                                 // 4. 太大就分裂：右半块挂回低一档
    struct buddy_node *right = (struct buddy_node *)((uint64_t)left
        + (1ULL << (i - 1)) * PAGE_SIZE);           // 右半块地址 = 左半块 + 半块大小
    /* 把 right 插进 free_list[i-1] 链表头 */
    i--;
}
alloc->page_order[offset / PAGE_SIZE] = order;      // 5. 户籍表登记阶数
```

**注意**：链表节点就存在空闲块的**开头几个字节里**（`struct buddy_node` 直接 cast 到 VRAM 地址上）——空闲块自己就是节点，不额外 malloc，这跟内核把管理结构塞进 struct page 是同一个思路。你可以思考：如果显存里的数据被破坏，链表会怎样？

**释放（找 buddy → 合并 → 递归）：[buddy.c:100-137](gpgpu_driver/lib/src/buddy.c#L100-L137)**

```c
int order = alloc->page_order[offset / PAGE_SIZE];              // 1. 查户籍
uint64_t buddy_offset = offset ^ ((1ULL << order) * PAGE_SIZE); // 2. 算 buddy 地址
/* 3. 在 free_list[order] 线性查找 buddy 是否空闲 */
/* 4a. 不在 → 自己插回链表  4b. 在 → 摘掉 buddy，合并，递归往上一档继续 */
```

**`offset ^ block_size` 就是算 buddy 的魔法公式**：兄弟块的地址只有"第 order 位"不同，异或一下就翻转那一位。这是 buddy 算法最优雅的一行代码，面试写白板题必考，务必亲手推一遍（比如 order=2、block=16KB，offset=0x30000 → buddy=0x34000）。

**slab：[slab_cache.c:6-35](gpgpu_driver/lib/src/slab_cache.c#L6-L35) + [slab_cache.c:56-78](gpgpu_driver/lib/src/slab_cache.c#L56-L78)**

```c
int slab_grow(struct slab_cache *cache) {
    uint64_t offset = buddy_alloc(cache->buddy, PAGE_SIZE);  // 1. 从 buddy 拿一页
    cache->buddy->page_order[offset / PAGE_SIZE] = cache->marker;  // 2. 户籍表写 marker！
    for (int i = 0; i < cnt; i++) {                          // 3. 把页切成 slot 串进 free_list
        struct slab_node *node = (struct slab_node *)(page_vaddr + i * cache->object_size);
        node->next = cache->free_list;
        cache->free_list = node;
    }
}
uint64_t slab_alloc(struct slab_cache *cache) {
    if (!cache->free_list) { if (slab_grow(cache) == -1) return -1; }  // 没了就再要一页
    offset = (uint64_t)cache->free_list - cache->buddy->vram_base;     // 头摘一个 slot
    cache->free_list = cache->free_list->next;
    ...
}
```

slab 页的 slot 链同样"自己存自己"（slot 的前 8 字节当下一个指针）。释放时 [slab_free](gpgpu_driver/lib/src/slab_cache.c#L80-L143) 还做了**缩容（shrink）**：一页的所有 slot 都空、且 slab 还有其他页时，把这页还给 buddy——没有这个机制，峰值用过的页就永远占着。

**面试加分点**：讲清楚 `page_order[]` 的双重身份（阶数 vs slab marker）和 slot 内嵌链表的设计，基本就证明你吃透了分配器。

#### 5.3.3 动手实验（本机即可，不依赖任何设备）

1. [test_buddy.c](gpgpu_driver/lib/test/test_buddy.c) 和 [test_slab.c](gpgpu_driver/lib/test/test_slab.c) 是纯 C 测试，可以直接本机编译跑（自己写 Makefile 或 gcc 直接编）。
2. 写一个 `buddy_dump()` 函数：遍历 21 条链表打印所有空闲块，然后在 test 里随机分配/释放 10000 次并每次校验 dump 的块数和大小守恒（总空闲字节数守恒 = 4GB - 已分配）。
3. 亲手推演：分配 100KB → 分裂路径画出来；释放后合并路径画出来。
4. 试试在 3.5GB 显存上跑初始化，看它切成哪些块（验证非 2 幂支持）。

#### 5.3.4 面试思考题

1. 分配 100KB 实际得到多大块？浪费多少？这种浪费叫什么碎片？
2. 本项目 buddy_free 找 buddy 用线性遍历链表，复杂度多少？内核怎么优化的？
3. 如果连续分配再交错释放，出现"总空闲 2GB 但分配不了 1.2GB"的极端情况，为什么？怎么缓解？
4. slab 的 shrink 逻辑里，怎么判断"整页都空"？为什么要限定"还有其他页"才还？
5. `page_order` 数组为什么能复用成 marker？这样做的代价是什么（提示：一个页被 slab 占用后还能被 buddy 误判吗）？

---

### 5.4 M4：GPU 汇编算子（⭐⭐⭐⭐⭐ 硬骨头，学 3-4 天）

#### 5.4.1 先讲人话：这些 .S 文件是"在显卡上跑的程序"

普通程序跑在 CPU 上，这些 kernel 跑在 QEMU 模拟的 GPU 核心上。它们不是 CUDA/OpenCL，而是**手写的 RV32 汇编**。每个 kernel 是一个独立的、从 `_start` 开始、以 `ebreak` 结束的小程序，被用户态加载进显存后，通过 ioctl 告诉设备"去显存 0xXXX 处开始执行"。

**编程模型（SIMT 的概念）**：一次 launch 会启动 grid（网格）个 block，每个 block 有若干 warp，每个 warp 固定 32 个 lane（线程）。本项目约定：**一个 block 一个 warp = 32 个线程**，所有线程执行**同一份代码**，只是各自的 `mhartid`（硬件线程号）不同——线程靠这个号码算出"我该处理哪份数据"。

**概念解释 —— mhartid**：RISC-V 里每个硬件线程（hart）有编号，用 `csrr t0, mhartid` 读。本项目把它编码成三层：[gpgpu_drv 无关，这是 QEMU 侧约定] `[31:13] block_id | [12:5] warp_id | [4:0] lane_id`。kernel 读到它后：

```text
lane_id  = mhartid & 0x1F      （低 5 位）
block_id = mhartid >> 13       （高 19 位）
线程线性编号 = block_id * 32 + lane_id
```

这等价于 CUDA 里的 `blockIdx.x * blockDim.x + threadIdx.x`。

#### 5.4.2 编译工具链：[kernels/Makefile](gpgpu_driver/kernels/Makefile)

```makefile
CROSS := riscv64-unknown-elf
ARCH  := rv32im_zicsr_zifencei_f     # RV32 + 整数乘除 + CSR + F 单精度浮点
ABI   := ilp32f
CFLAGS := -march=$(ARCH) -mabi=$(ABI) -nostdlib -Wl,-Ttext=0x0
%.bin: %.elf
	$(OBJCOPY) -O binary $< $@       # ELF 剥成纯二进制指令流
%.dump: %.elf
	$(OBJDUMP) -d $< > $@            # 反汇编给人看
```

三步流程：汇编 → ELF → 剥成裸二进制（.bin 是设备真正执行的，设备解释器从 offset 0 开始取指）。`.dump` 是调试利器：伪指令（`li`/`mv`/`beqz`）展开成什么、指令编码对不对，全看它。

#### 5.4.3 精读三个代表性 kernel

**① gemm.S（矩阵乘，最简单直白的并行模式）——[gemm.S 全篇 58 行](gpgpu_driver/kernels/gemm.S)**

```asm
_start:
    lw   a1,  0(a0)       # a1 = a_off（参数都从 a0 指向的 args 块里按偏移取）
    lw   a2,  4(a0)       # a2 = b_off
    lw   a3,  8(a0)       # a3 = c_off
    lw   a4, 12(a0)       # a4 = M
    lw   a5, 16(a0)       # a5 = K
    lw   a6, 20(a0)       # a6 = N

    csrrs t6, mhartid, x0
    andi  t1, t6, 0x1F          # t1 = lane_id
    srli  t6, t6, 13            # t6 = block_id
    slli  t6, t6, 5             # block_id * 32
    add   t0, t6, t1            # t0 = 全局线程编号
    mul   t5, a4, a6
    bge   t0, t5, done_oob      # 线程数多于输出元素数 → 越界线程直接退出

    div   t1, t0, a6            # row = 编号 / N
    rem   t2, t0, a6            # col = 编号 % N
    ...
loop:                           # 每个线程算 C[row][col] = Σ A[row][k] * B[k][col]
    flw   ft0, 0(t4)            # 取 A[row][k]
    flw   ft1, 0(t5)            # 取 B[k][col]（B 已被 dump 工具转置成 [K,N] 布局！）
    fmadd.s ft3, ft0, ft1, ft3  # acc += a*b（融合乘加，一条指令）
    addi  t0, t0, 1
    j     loop
done:
    fsw   ft3, 0(t5)            # 写回 C[row][col]
done_oob:
    ebreak                      # 本线程执行完毕，通知设备
```

看懂这 58 行，你就懂了"GPU 编程 = 让成千上万个线程各自算一小块"。**参数传递约定**：kernel 不靠函数调用传参，而是约定 `a0 = 参数块地址`（该地址由 ioctl 的 kernel_args 寄存器给出，指向显存里的一块 args buffer），然后按固定偏移 `lw`。**这意味着 C 侧 args 结构体的字段顺序和汇编里 lw 顺序必须严格一致**——错了不 crash，只是算出错误结果，这类 bug 极难查（README 原话）。

**② rmsnorm.S（第一次出现"线程间通信"）——[rmsnorm.S 全篇 70 行](gpgpu_driver/kernels/rmsnorm.S)**

RMSNorm 的数学：`out = x / sqrt(mean(x²) + eps) * gamma`。关键在 `mean(x²)` 是**全 896 个元素的平均**，而 32 个线程每人手里只有自己算的 28 个元素的平方和——必须把 32 份局部和**归约（reduce）**成一个总和，人人拿到。

```asm
    VSHFL_XOR 2, 0, 1          # 拿 lane_id^1 号线程的 ft0 到自己的 ft2
    fadd.s ft0, ft0, ft2
    VSHFL_XOR 2, 0, 2          # 拿 lane_id^2 的
    ...
    VSHFL_XOR 2, 0, 16         # 5 步后，ft0 = 全体 32 个 lane 的和
```

**概念解释 —— shuffle**：真实 GPU 里线程之间交换寄存器数据叫 warp shuffle（CUDA 的 `__shuf_xor_sync`）。本项目在 QEMU 设备侧自定义了一条 `vshfl` 指令（模拟 GPU shuffle），汇编里用 `.word` 手工拼指令编码：

```asm
.macro VSHFL_XOR rd, rs1, mask
    .word (\mask << 20) | (\rs1 << 15) | (\rd << 7) | 0x0B
.endm
```

`0x0B` 是这条自定义指令的 opcode（RV32 里 0x0B 是 custom-0 指令空间，厂商/研究项目可以自己定义）。5 步树形归约：先和相隔 1 的换，再隔 2、4、8、16，log2(32)=5 步完成全体求和。**这是面试讲"自定义指令扩展"的绝佳素材**。

**③ softmax_decode.S（本项目最复杂 kernel + 一个真实 bug 的故事）——[softmax_decode.S 全篇 108 行](gpgpu_driver/kernels/softmax_decode.S)**

softmax 需要三趟扫描：第一趟找最大值（防指数溢出，`exp(x-max)`），第二趟算 `exp(x-max)` 并累加和，第三趟除以和。每趟都要跨 lane 归约（max 用 `fmax.s`、sum 用 `fadd.s`）。

**真实 bug 故事**（docs/5.23 开发日志）：当 `S < 32` 时（序列还短，只有 S 个有效分数），有 `32-S` 个"空 lane"没有活干。原代码让空 lane 直接跳到 done，**跳过了累加器初始化**（`ft0 = -INF`、`ft6 = 0`），但后面的 VSHFL 归约是无条件执行的——空 lane 的脏寄存器值污染了归约结果。表现：S=2,4,5,6... 大量 FAIL，而 S=2 的幂恰好 PASS（脆弱巧合）。修复：**空 lane 也必须把累加器初始化成"归约单位元"**（max 的单位元是 -INF，sum 的单位元是 0），让它们参与归约但不影响结果。

**这个故事的价值**：面试讲 bug 定位（密集采样暴露 → 分析空 lane 路径 → 单位元修复 → 误差从 1e0 降到 1.43e-6），比讲十个知识点都有说服力。

**自定义指令之二**：`VEXP_S`（[softmax_decode.S:3-5](gpgpu_driver/kernels/softmax_decode.S#L3-L5)）——设备解释器直接用 host 的 `expf()` 实现，一条指令完成指数计算。

**④ 其他 kernel 一句话清单**（都值得过一遍）：

| kernel | 作用 | 值得注意的点 |
|---|---|---|
| [embedding_lookup.S](gpgpu_driver/kernels/embedding_lookup.S) | 按 token id 取 embedding 行 | 查表 = 地址计算，无循环 |
| [rope.S](gpgpu_driver/kernels/rope.S) | 旋转位置编码 | sincos 表的交错布局 |
| [qkt_decode.S](gpgpu_driver/kernels/qkt_decode.S) | Q·Kᵀ 注意力分数 | 2 KV head 被 14 Q head 复用（GQA） |
| [pv_decode.S](gpgpu_driver/kernels/pv_decode.S) | 概率加权 V | 累加 + 乘 |
| [silu.S](gpgpu_driver/kernels/silu.S) / [vmul.S](gpgpu_driver/kernels/vmul.S) | FFN 激活 | silu 公式 `x*sigmoid(x)` |
| [argmax.S](gpgpu_driver/kernels/argmax.S) | 从 151936 个 logits 找最大 | 15 万元素的两趟扫描 + 归约 |
| [broadcast_add.S](gpgpu_driver/kernels/broadcast_add.S) | 加 bias/残差 | 一维广播 |

#### 5.4.4 动手实验

1. 装好 `riscv64-unknown-elf-gcc` 后：`cd kernels && make gemm.dump`，对照 .dump 验证 `li t0, 0xff800000`（-INF 的位模式）展开成几条指令。
2. 写自己的 `vec_scale.S`：`out[i] = in[i] * scale`，编译出 .bin，照着现有测试写个 host 参考对比。
3. 手算：softmax_decode 里 `iter_count = ceil((S - lane_id) / 32)` 这条汇编（`sub/addi 31/srli 5`）为什么等价于向上取整除 32。

#### 5.4.5 面试思考题

1. 为什么 gemm 的 B 矩阵要提前转置成 [K,N]？不转置会怎样？
2. `fmadd.s` 和先 `fmul` 再 `fadd` 有什么区别（精度/性能）？
3. 树形归约为什么是 5 步？如果 warp 是 64 线程要几步？
4. 什么是"线程分歧（divergence）"？min-PC 调度是什么、有什么缺陷（README 设计笔记里有答案）？
5. `.word` 手工拼指令 vs 汇编器自动编码，各自用在什么场景？本项目为什么用 `.word`？

---

### 5.5 M5：权重加载器 weight_loader（⭐⭐⭐ 学 2 天）

#### 5.5.1 先讲人话

Qwen2-0.5B 模型就是一堆浮点数参数（总共 2.35GiB），存在磁盘上一组 `.bin` 文件里。推理时这些参数必须在显存里（kernel 直接从显存读权重做计算）。weight_loader 的活：**把文件读出来、按固定布局摆进显存、给每个张量（tensor）记好显存地址**，之后谁要哪层的哪个权重，来查表就行。

#### 5.5.2 代码导读

**布局契约：[weight_layout.h](gpgpu_driver/lib/include/weight_layout.h)** —— 本项目 C 侧和 Python 工具侧共享的"地图"。每层 24 层结构完全相同，单层内每个张量的 offset 是硬编码常量：

```c
#define WL_HIDDEN_SIZE 896
#define WL_OFF_INPUT_NORM 0                  // 输入归一化权重在层文件开头
#define WL_OFF_Q_PROJ_W 3584                 // 紧接着 q_proj 权重（896×4 字节后）
#define WL_LAYER_BYTES 59649536              // 单层总字节 ≈ 57MB
_Static_assert(WL_OFF_DOWN_PROJ_W + WL_SIZE_DOWN_PROJ_W * 4 == WL_LAYER_BYTES,
               "weight_layout.h: layer offsets sum mismatch with WL_LAYER_BYTES");
```

**两个值得讲的工程实践**：
1. `_Static_assert`（编译期断言）把"offset 之和必须等于层大小"这种契约写死在编译期——谁改错立即编译失败。这比运行时 assert 强得多。
2. 所有 `*_proj.weight` 在 dump 阶段就被 Python 转置成 gemm.S 直接吃的 [K,N] 布局——**把"转置"这个活从热路径挪到离线工具**，kernel 代码因此简单。

**加载主流程：[weight_loader.c:150-299](gpgpu_driver/lib/src/weight_loader.c#L150-L299)** 六步：

```text
1. 按 8 个 region 计算每个需要多少字节
2. stat() 校验全部 27 个文件的尺寸（文件损坏 → 立即失败，不带病往下走）
3. calloc 分配 loader 结构
4. 8 次 gpuMalloc 申请显存
5. 在每个 region 内算 embedding/lm_head/final_norm/各层 base 地址
6. fread 整文件到 host buffer → gpuMemcpy H2D 进显存
```

**8 个 region 的由来（本项目最精彩的工程决策之一，[weight_loader.c:27-44](gpgpu_driver/lib/src/weight_loader.c#L27-L44) 注释里写满了）**：

- 最初方案：把 2.35GiB 权重当一整块 `gpuMalloc` 一次 → buddy 会把它 round-up 到 4GB，直接撑爆显存。
- 4 块方案：每块 683MB → round-up 到 1GiB，4 块共占 3.95GiB，还是超。
- **最终 8 块方案**：embedding 一块、lm_head 一块、每 4 层一块（228MB → round 到 256MB），总共 3.5GiB，给 scratch/KV cache 留出 500MB。

**这个故事的价值**：它展示了"分配器行为（round-up）倒逼上层数据布局设计"的完整链条。面试讲出来，证明你理解内存分配器不是黑盒，其行为会影响整个系统的资源规划。

**校验先行**：[check_file_size](gpgpu_driver/lib/src/weight_loader.c#L93-L106) 在分配前 stat 所有文件——顺序设计很讲究：**先验证输入、再申请资源**，避免"显存都申请了才发现文件坏"。

#### 5.5.3 动手实验

1. 用计算器/Excel 验算 weight_layout.h 每个 offset：如 `WL_OFF_Q_PROJ_W = 896×4 = 3584 ✓`，`WL_OFF_Q_PROJ_B = 3584 + 896×896×4 = 3214848 ✓`。
2. 读 [tools/dump_weights.py](gpgpu_driver/tools/dump_weights.py)，找出它 transposes 权重的代码段（搜 `transpose` 或 `contiguous`）。
3. 手推 8 region 的内存账单：每块请求字节 → round 到哪个 order → 实际占用 → 总计。

#### 5.5.4 面试思考题

1. 为什么 weight_loader 不做 lazy loading（用到哪层再加载）？
2. 加载中途失败，fail 路径怎么保证显存不泄漏？[weight_loader.c:290-298](gpgpu_driver/lib/src/weight_loader.c#L290-L298) 找答案。
3. README 说 VM 里权重文件曾和 host 不一致，表现是什么、怎么防？（checksum gate）

---

### 5.6 M6：KV cache（⭐⭐⭐ 学 1-2 天）

#### 5.6.1 先讲人话

Transformer 生成第 N 个 token 时，要"看"前面 N-1 个 token 的内容。朴素做法是每次把整个历史重算一遍，O(N²) 慢到不可接受。KV cache 的思路：**算过的 K/V 存起来，下次直接用**，每个新 token 只算自己那一行 K/V 然后追加进缓存。

#### 5.6.2 代码导读

**一次性预分配：[kv_cache.c:7-37](gpgpu_driver/lib/src/kv_cache.c#L7-L37)**

```c
size_t total_bytes = num_layers * num_kv_heads * max_seq * head_dim * sizeof(float);
kv_cache->K_dev_off = gpuMalloc(ctx, total_bytes);   // 24×2×128×64×4 = 1.5MB
kv_cache->V_dev_off = gpuMalloc(ctx, total_bytes);
```

K 和 V 各一块大显存，layout 是 `[layer][head][seq][dim]` 四维。本项目一次申请到位、不回收，是 toy 设计（真实推理引擎有 paged attention 等复杂管理，面试可提一句）。

**两段式追加协议（append + commit）：[kv_cache.c:39-81](gpgpu_driver/lib/src/kv_cache.c#L39-L81)**

```c
int kv_cache_append_layer(ctx, cache, layer, K_new, V_new) {
    if (cache->filled >= cache->max_seq) return -1;      // 满
    if (layer != cache->pending) return -2;              // 必须按 0,1,2...23 顺序追加
    ... /* 把这个 token 第 layer 层的 K/V 写到 filled 行 */
    cache->pending++;                                    // 完成一层
}
int kv_cache_commit_token(kv_cache_t *cache) {
    if (cache->pending != cache->num_layers) return -2;  // 24 层没齐不能提交
    cache->pending = 0;
    cache->filled++;                                     // 提交：推进水位
}
```

**为什么拆两段**：一个 token 要过 24 层，每层都会产生 K/V。如果每层 append 完立刻把 `filled++`，中间任何一层出错，缓存里就混进了"半截 token"，下次推理读到的历史是坏的。两段式协议保证 **filled 永远只指向完整 token 的边界**——要么 24 层全 append 完才提交，要么整个 token 都不算数。这是"事务性"思想在内存管理里的最小实现，讲清楚很加分。

**寻址 getter 复用：[kv_cache.c:83-99](gpgpu_driver/lib/src/kv_cache.c#L83-L99)** —— `kv_cache_K_off/V_off` 集中封装 stride 算式，开发日志"原则 14"专门讲：重复的 stride 算式必须靠 getter 复用，不能到处手抄（抄三处必有一处错）。

#### 5.6.3 动手实验

1. 手推寻址公式：layer=5、head=1、pos=10 的 K 元素在显存哪个偏移？
2. 读 [transformer.c:697](gpgpu_driver/lib/src/transformer.c#L697) 附近的 `kv_append_from_device`，理解"hybrid 路径"：device 的 K/V 先 D2H 拷回 host，host 调 append_layer 再 H2D 写回缓存——数一下一个 token 的 24 层要多少次跨设备往返（48 次！这就是性能瓶颈）。

#### 5.6.4 面试思考题

1. 如果第 12 层 append 失败，前面 11 层的 pending 数据怎么办？下次 append 会怎样？
2. 本项目 K/V 走 D2H→H2D 绕路，真实 GPU 上应该怎么做？（device-side append kernel）
3. `max_seq=128` 意味着什么限制？满了之后 append 返回什么？

---

### 5.7 M7：Transformer 推理层（⭐⭐⭐⭐ 学 2-3 天）

#### 5.7.1 先讲人话

前面的模块都是"零件"，transformer.c 是"总装车间"：它不发明任何算子，只是按 Qwen2 模型的数学结构，把 20+ 个 kernel launch **按正确顺序、用正确参数、把中间结果放在正确 buffer 里**串成流水线。读这个模块不需要懂 AI 数学，当成"一个巨大的状态机编排器"读即可。

#### 5.7.2 代码导读（从外到内四层）

**第 1 层：资源准备 `transformer_init`（[transformer.c:371](gpgpu_driver/lib/src/transformer.c#L371)）**

对照 [transformer.h:78-134](gpgpu_driver/lib/include/transformer.h#L78-L134) 的 workspace 结构：一次性 `gpuMalloc` 9 块 scratch buffer（RESID_A/B、SCRATCH_X/Q/K/V/SCORES/FFN_A/FFN_B）、logits、sincos 表、token 缓冲，然后把 11 个 kernel 的 .bin 文件加载进显存（`load_kernel_bin`），再给每个 kernel 分配 args buffer。**所有地址都记在 workspace 这个"地址簿"里**，后面 op 只管引用。

**第 2 层：算子封装 `op_*`（[transformer.c:754-914](gpgpu_driver/lib/src/transformer.c#L754-L914)）**

每个 op 函数做三件事：填本算子的 args 结构体（写进显存里的 args buffer）→ 算 grid/block → `launch_op` 调 ioctl。看 [op_qkt_decode:865-885](gpgpu_driver/lib/src/transformer.c#L865-L885)：

```c
struct qkt_decode_args args = {
    .Q_off = Q_off, .K_cache_off = K_cache_off, .scores_off = scores_off,
    .cur_pos = cur_pos, .max_seq = ws->cfg.max_seq, .head_dim = ws->cfg.head_dim,
    .q_per_kv = ws->cfg.num_q_heads / ws->cfg.num_kv_heads,   // 14/2 = 7（GQA）
    .scale_bits = sf.u,    // float 的位模式当 u32 传，设备侧不转类型直接比位
};
uint32_t grid[3] = {ws->cfg.num_q_heads, 1, 1}, block[3] = {32, 1, 1};  // 14 block × 1 warp
```

**注意 `union { float f; uint32_t u; }`**：C 里 float 传给"只认 32 位整数的寄存器"时，用 union 取其位模式传递，避免隐式类型转换——和 M4 里"FP 寄存器存 bit pattern"是同一个世界观的细节。

**grid 计算**（[transformer.c:174](gpgpu_driver/lib/src/transformer.c#L174) 附近的 `calc_dispatch_1warp_per_block`）：`total` 个线程 = `ceil(total/32)` 个 block，每 block 1 warp。比如 gemm 输出 151936 个 logits（lm_head），就要 4748 个 block。

**第 3 层：单层解码器 `transformer_decoder_layer`（[transformer.c:934](gpgpu_driver/lib/src/transformer.c#L934)）**

24 层共用这一个函数。每层的 20+ 步流水（对照 [transformer.c:951](gpgpu_driver/lib/src/transformer.c#L951) 起）：

```text
attention 半区：
  input_norm(rmsnorm) → q_proj/k_proj/v_proj(gemm) → bias(broadcast_add)
  → q/k rope → KV cache append → qkt_decode → softmax_decode
  → pv_decode → o_proj(gemm) → residual add
FFN 半区：
  post_norm(rmsnorm) → gate_proj/up_proj(gemm) → silu → vmul → down_proj(gemm)
  → residual add
```

**scratch buffer 复用**是本模块精髓：SCRATCH_X 一会儿是 rmsnorm 输出、一会儿是 attention 输出、一会儿是 down_proj 输出（[transformer.h:89-91](gpgpu_driver/lib/include/transformer.h#L89-L91) 注释写明）。显存只有 4GB，权重占 3.5GiB，中间结果必须反复覆盖使用——**这是手写推理引擎的真实状态，比教科书里"每层新分配"干净得多，但可读性差，面试官问"为什么复用"时你能讲出内存账单**。

**第 4 层：完整一步 `transformer_step`（[transformer.c:1215-1276](gpgpu_driver/lib/src/transformer.c#L1215-L1276)）**

```text
1. H2D 写当前 token id
2. embedding_lookup → RESID_A
3. for layer 0..23: decoder_layer      ← 每层 20+ 次 kernel launch，共 ~500 次
4. final rmsnorm
5. lm_head gemm → LOGITS（151936 个分数）
6. argmax → 下一个 token id
7. KV cache commit（24 层全 append 完才提交）
8. D2H 读回下一个 token id 返回
```

**一个 token 要 launch 大约 500 次 kernel**（24 层 × 每层 ~20 个算子）。每次 launch 都是一次 ioctl + 一次等待 100ms 中断——所以推理慢是必然的，但也正因如此，链路里的每一环都被反复执行验证。性能优化的方向（kernel fusion、一次 launch 跑多层）都在 README 路线图里。

**调试钩子**：[transformer.h:181-213](gpgpu_driver/lib/include/transformer.h#L181-L213) 的 `DUMP_SCRATCH` 宏——`#ifdef DUMP_INTERMEDIATES` 编译期开关，打开后每一步中间结果都写到 `dump/step_NN/layer_MM_name.bin`，和 host 端 PyTorch 参考实现逐位对比。这是本项目能端到端调通的工程基石。

#### 5.7.3 动手实验

1. 画一张 A3 大小的"一个 token 的完整执行流水图"：从 embedding 到 argmax，标出每一步的输入 buffer → 算子 → 输出 buffer（用箭头画 buffer 流转，你会直观看到 SCRATCH_X 被复用了 3+ 次）。
2. 数一数一个 token 共 launch 多少次 kernel（精确值），再估算一次推理的总时间 = 次数 × 100ms。
3. 读 [infer_tokens.c:232-262](gpgpu_driver/lib/test/infer_tokens.c#L232-L262)，理解 prefill（把 prompt 当成一串 single-token step 依次喂）和 decode 的区别。

#### 5.7.4 面试思考题

1. 为什么 prefill 可以用 single-token step 实现？这样做的代价是什么？
2. argmax 的结果怎么从 device 传回 host？（token_id_buf D2H）为什么不等中断就 D2H 读会出错？
3. KV cache 的 commit 放在 argmax 之后、D2H 之前（[transformer.c:1263-1267](gpgpu_driver/lib/src/transformer.c#L1263-L1267)），为什么放在这个位置？
4. 如果某一步 op 返回错误码，transformer_step 会不会提交 KV cache？找出代码依据。

---

### 5.8 M8：测试与验证体系（⭐⭐⭐⭐ 学 2-3 天）

#### 5.8.1 先讲人话

驱动/算子项目最大的敌人是"看起来能跑但结果错了一点"。这个项目用一套**对照测试方法**武装自己：每个 GPU 算子在 host 上用 C 写一份"标准答案"（reference），跑完 device 算子和 reference 对比，误差超过阈值就 FAIL。更狠的是 `DUMP_INTERMEDIATES`：把推理中间每一步的结果都 dump 出来，和 PyTorch 官方实现逐位对比，精确定位"哪一层开始错"。

#### 5.8.2 方法论三要素（从测试代码里提炼）

1. **独立 reference**：[test_gemm.c](gpgpu_driver/lib/test/test_gemm.c) 用 host C 循环实现 gemm，device 跑汇编版，两者对比。**reference 必须独立于被测实现**（用不同的人/不同语言写），否则错误会复制。
2. **边界采样**：docs/5.23 的教训——测试矩阵恰好采到 2 的幂全 PASS，密集采样 0~39 才暴露 bug。**测试集要覆盖边界和非 2 幂**。
3. **误差阈值**：softmax 修复后 `max_err ≤ 1.43e-6 (≤ 4 ULP)`。FP32 运算顺序不同会引入舍入误差，阈值要按算法合理设定，而不是无脑 `==`。

#### 5.8.3 代码导读

- [lib/test/](gpgpu_driver/lib/test/) 30 个测试文件，一个算子一个，命名规整：`test_<算子名>.c`。每个都遵循"构造输入 → device 跑 → host 参考 → 对比"四步。
- [tools/check_layer.py](gpgpu_driver/tools/check_layer.py)：逐层 dump 对照工具（配合 DUMP_INTERMEDIATES 的产物）。
- [ref/](gpgpu_driver/ref/)：官方 reference dumps（step_00/step_01 已入库，其余可生成）。
- [test_transformer](gpgpu_driver/lib/test/)（Makefile 目标）：端到端 sanity test，期望输出 `prefill last (expected 3): 3 PASS`、`decode step 0 (expected 323): 323 PASS`——注意这里的 3 和 323 是"固定的标准 token id"，跟 PyTorch 官方输出一致，说明整条链路数学上完全正确。

#### 5.8.4 动手实验

1. 通读 [test_softmax.c](gpgpu_driver/lib/test/test_softmax.c)（3800 行里最小的一批），画出它的四步结构。
2. 给 M4 里你写的 `vec_scale.S` 写一个 test（四步结构照抄）。
3. 读 docs/5.9 开发日志的"原则"章节（每篇日志末尾都有"新增原则"），把 15 条原则抄成清单——这些就是本项目作者沉淀的工程智慧，面试引几条很加分。

#### 5.8.5 面试思考题

1. 为什么不用 `==` 比较浮点结果？ULP 是什么？
2. "测试全 PASS"能证明实现正确吗？结合 5.23 的 2 幂巧合说。
3. 如果端到端推理结果错了一个 token，你会怎么定位？（提示：checksum → 逐层 dump → 二分定位）

---

## 6. 项目简历包装方案

> 诚实原则（重要）：这是别人开源的项目，**简历上不能写"独立开发"**。但"复现 + 深挖 + 改进 + 测试"是完全可以写的真实经历。建议你在学习过程中至少完成 6.3 节列的 1-2 个小改进，让"我的贡献"部分有真凭实据（代码 diff 就是证据）。

### 6.1 简历项目描述模板（两个方向）

#### 版本 A：嵌入式 Linux 驱动方向（突出内核功底）

> **基于 QEMU 虚拟 PCI GPU 的 Linux 驱动及加速器软件栈开发**
> （GitHub 开源项目 gpgpu-driver 的深入研读与二次开发，个人独立复现并改进）
>
> - 研读并复现了面向 QEMU 虚拟 PCI GPGPU 设备的 Linux PCI 字符设备驱动：实现设备 probe/remove、BAR0/BAR2/BAR4 资源映射（`pci_iomap`）、Legacy INTx 中断注册、`/dev/gpgpu` 字符设备注册的完整流程；
> - 设计了基于 `mmap` + `ioctl` 的用户态加速器接口：通过 `io_remap_pfn_range` 将 VRAM BAR 直接映射到用户态实现零拷贝访问，通过 `GPGPU_IOC_DISPATCH` 命令实现 kernel dispatch，使用等待队列 + 中断实现内核执行完成的进程睡眠/唤醒同步；
> - 独立实现了管理 4GB VRAM 的用户态 buddy 伙伴分配器与 6 档 slab 缓存（参考 Linux 内核内存管理设计），解决了大块权重分配 round-up 导致的显存碎片问题；
> - 手写 20+ 个 RISC-V RV32 SIMT 汇编算子（GEMM、RMSNorm、Softmax、RoPE、SiLU 等），通过 `.word` 手工编码扩展自定义指令（vexp、vshfl），理解并实现 warp 级线程通信与树形归约；
> - 在自研驱动上完整跑通 Qwen2-0.5B（24 层 Transformer、2.35GiB FP32 权重）的 token 级推理，建立 bit-exact 逐层 dump 对照测试体系定位并修复算子边界 bug。

#### 版本 B：BSP 方向（突出软硬件协同）

> **QEMU 虚拟 GPU 平台 BSP 适配与驱动开发**
> （开源项目 gpgpu-driver，个人独立复现并二次开发）
>
> - 完成 RISC-V Linux 平台下虚拟 PCI 显卡设备的 BSP 适配：交叉编译环境搭建、内核驱动移植（RISC-V 平台 PLIC 不支持 MSI-X，适配 Legacy INTx 中断方案）、设备树/PCI 枚举验证；
> - 分析并适配 QEMU 设备侧（PCI BAR 定义、MMIO 控制寄存器组、kernel dispatch 协议、中断触发时序），编写用户态测试程序验证 MMIO 读写、显存映射与中断完成机制；
> - 基于 `io_remap_pfn_range` 实现 BAR2 显存的用户态映射，设计 `vm_pgoff` 多 BAR 区分方案；优化驱动 probe 错误处理路径与 ioctl 并发安全；
> - 实现 VRAM buddy/slab 混合分配器，针对 2.35GiB 权重加载场景设计 8-region 布局方案，将显存占用从 4GB 降至 3.5GB；
> - 编写 30+ 个算子级对照测试与端到端 sanity 测试，建立 host/VM 权重 checksum 校验流程，定位并修复显存数据一致性问题。

### 6.2 技术亮点提炼（面试口述素材，每条都有代码依据）

| # | 亮点 | 一句话口述 | 代码依据 |
|---|---|---|---|
| 1 | 中断 + 等待队列完成同步 | "ioctl 提交任务后进程睡眠，中断 handler 置位唤醒，这是 GPU 驱动任务完成通知的最小实现" | [gpgpu_drv.c:76-81](gpgpu_driver/gpgpu_drv.c#L76-L81)、[gpgpu_drv.c:150-152](gpgpu_driver/gpgpu_drv.c#L150-L152) |
| 2 | mmap 零拷贝显存访问 | "显存映射到用户态后，memcpy 就能读写显存，省掉 read/write 两次拷贝" | [gpgpu_drv.c:92-112](gpgpu_driver/gpgpu_drv.c#L92-L112)、[libgpgpu.c:168-179](gpgpu_driver/lib/libgpgpu.c#L168-L179) |
| 3 | `vm_pgoff` 区分多 BAR | "mmap 的 offset 被复用为 BAR 选择器，0=控制寄存器、1=显存，踩过映射错 BAR 的坑" | [gpgpu_drv.c:97-106](gpgpu_driver/gpgpu_drv.c#L97-L106) |
| 4 | 分配器对抗 round-up | "buddy 按 2 的幂取整，2.35GB 权重若整体申请会 round 到 4GB 爆显存，拆 8 个 region 后总占用 3.5GB" | [weight_loader.c:27-44](gpgpu_driver/lib/src/weight_loader.c#L27-L44) |
| 5 | `page_order` 双重身份 | "一个每页 1 字节的数组既存 buddy 阶数又存 slab 类型标记，free 时靠它路由到正确释放路径" | [libgpgpu.c:156-165](gpgpu_driver/lib/libgpgpu.c#L156-L165) |
| 6 | 空 lane 归约单位元 bug | "softmax 在序列长度 <32 时，空线程跳过初始化导致脏寄存器污染跨 lane 归约，修复方案是把累加器初始化为归约单位元（-INF/0）" | [softmax_decode.S:34-49](gpgpu_driver/kernels/softmax_decode.S#L34-L49)、docs/5.23 |
| 7 | 自定义指令扩展 | "用 .word 在 RV32 custom-0 指令空间手工编码了 vexp（指数）和 vshfl（warp shuffle）两条扩展指令，支撑 softmax 和树形归约" | [rmsnorm.S:2-4](gpgpu_driver/kernels/rmsnorm.S#L2-L4)、[softmax_decode.S:3-5](gpgpu_driver/kernels/softmax_decode.S#L3-L5) |
| 8 | 事务式 KV cache 协议 | "K/V 追加和提交拆成两段，24 层全部追加成功才推进水位，保证缓存永远不包含半截 token" | [kv_cache.c:39-81](gpgpu_driver/lib/src/kv_cache.c#L39-L81) |
| 9 | 编译期契约校验 | "权重布局 offset 用 _Static_assert 在编译期校验总和，改错布局编译不过" | [weight_layout.h:75-82](gpgpu_driver/lib/include/weight_layout.h#L75-L82) |
| 10 | bit-exact 定位方法 | "每个算子 host 参考对照 + 逐层中间结果 dump，能精确定位第几层第几个算子出错" | [transformer.h:181-213](gpgpu_driver/lib/include/transformer.h#L181-L213)、[tools/check_layer.py](gpgpu_driver/tools/check_layer.py) |

### 6.3 个人贡献描述（诚实的三种写法）

**写法 1（最低配：深读复现）**：适用于学习完成后还没做改进的情况。
> "对项目进行了完整的代码研读与复现：逐行分析 PCI 驱动 probe/中断/mmap/ioctl 路径，独立复现 buddy/slab 分配器测试，手写补充了 XX 算子的单元测试。"

**写法 2（推荐：做 1-2 个真改进）**。以下是**难度递进的可选改进清单**，任选 1-2 个做掉，简历就完全站得住：

| 改进 | 难度 | 产出 | 面试价值 |
|---|---|---|---|
| 修复 ioctl 信号打断路径：`wait_event_interruptible` 返回 `-ERESTARTSYS` 时处理 kernel_done 状态 | ⭐⭐ | 一个 patch | 展示对中断/睡眠语义的理解 |
| 驱动 ioctl 加并发保护（互斥或 per-open 状态分离） | ⭐⭐⭐ | 一个 patch + 并发测试 | 展示对并发缺陷的嗅觉 |
| 中断 handler 增加"读 IRQ_STATUS 判断是否本设备中断"（配合 IRQF_SHARED 的正确用法） | ⭐⭐ | 一个 patch | 展示对共享中断语义的理解 |
| 修 probe 错误路径 bug（如未使用的 `err_request_irq_kernel` 标签/路径缺释放） | ⭐ | 一个 patch | 展示对资源管理的敏感度 |
| 给 buddy 分配器加 `buddy_dump()` 统计函数 + 随机分配释放守恒测试 | ⭐⭐ | 代码 + 测试 | 展示测试意识 |
| 写一个新 GPU 汇编 kernel（如 `vec_scale.S`）及其对照测试 | ⭐⭐⭐ | kernel + 测试 | 展示汇编/SIMT 能力 |
| 修 slab_free 里"meta 找不到就 NULL 解引用"的隐患（加防御） | ⭐ | 一个 patch | 展示防御式编程 |
| 把 KV cache 的 D2H/H2D 路径改造成 device-side append kernel（README 路线图第一项） | ⭐⭐⭐⭐⭐ | 较大 patch | 顶级加分，但工作量大 |

**写法 3（高配：完整二开）**：做完 3+ 个改进后，可以写"基于开源项目进行了 XX 方向的重构与增强"。

### 6.4 面试口述脚本

#### 2 分钟版（自我介绍里的项目段）

> "我深入研读并二次开发了一个基于 QEMU 虚拟 GPU 的 Linux 驱动项目。它用 QEMU 虚拟了一块 PCI 显卡，我完整复现了它的 Linux PCI 字符设备驱动：probe 时映射 BAR0/BAR2/BAR4、注册 /dev/gpgpu 字符设备和 Legacy INTx 中断；用户态通过 mmap 把显存直接映射进来做零拷贝访问，通过 ioctl 提交 kernel dispatch，驱动写 MMIO 寄存器启动设备执行，然后进程在等待队列上睡眠，设备执行完拉中断唤醒进程。这个项目让我完整走通了'用户态提交任务 → 驱动写寄存器 → 设备执行 → 中断通知 → 用户态继续'这条加速器驱动的核心链路。我还实现了显存的 buddy+slab 分配器、手写了 20 多个 RISC-V 汇编算子，最终在这个自研驱动上跑通了 Qwen2-0.5B 大模型的 token 级推理。项目过程中我修复了 ioctl 的信号打断问题和中断共享判断问题【替换成你实际做的改进】。"

#### 5 分钟版（面试官让"详细介绍项目"时）

结构建议：**总-分-坑**。

1. **总**（30 秒）：项目定位 + 软件栈图（口述 0.2 节那张图）。
2. **分**（2.5 分钟）：挑 3 个模块讲深——驱动（probe 流程 + mmap/ioctl 设计）、分配器（buddy/slab + 8-region 故事）、算子（gemm 汇编 + 树形归约 + 自定义指令）。
3. **坑**（1.5 分钟）：讲 2 个真实 bug 故事（softmax 空 lane 污染；权重文件不一致）。每个按"现象 → 定位过程 → 根因 → 修复 → 教训"讲。
4. **收**（30 秒）：主动说已知局限和下一步优化（KV cache device-side append、kernel fusion、并发保护），展示工程判断力。

### 6.5 面试官高频追问清单 + 参考答案

> 这些问题按"大概率被问"排序。先自己答，再看答案。答案要点化，面试时用自己的话展开。

**Q1：probe() 是谁调用的？调用时机？**
答：PCI 核心在枚举总线时调用。系统启动（或热插拔）时，PCI 总线扫描到设备，读出配置空间的 vendor/device ID，遍历所有注册的 `pci_driver`（本驱动通过 `module_pci_driver` 宏注册）的 id_table 匹配，命中就调 probe。匹配不上就什么也不发生。

**Q2：为什么 ioctl 里要用等待队列而不是 while 轮询 kernel_done？**
答：轮询是忙等，会 100% 占用 CPU 空转 100ms；等待队列让进程睡眠、让出 CPU，中断来了才唤醒，CPU 可以被别的任务用。这是驱动开发的基本原则：**用户进程上下文可以睡，中断上下文不能睡**，所以"睡在进程侧、醒来靠中断"是最优分工。

**Q3：中断 handler 里能做什么、不能做什么？本项目 handler 里做了什么？**
答：handler 运行在中断上下文：不能睡眠（没有进程上下文可睡）、不能调 copy_to/from_user（可能引发页错误睡眠）、应尽量短。本项目 handler 只置 `kernel_done=1` 并 `wake_up_interruptible`（唤醒不睡眠，安全），真正后续处理由被唤醒的进程继续。另外 handler 没读 IRQ_STATUS 判断中断来源，配合 IRQF_SHARED 这是隐患：共享中断线上别的设备触发中断也会调进这个 handler。

**Q4：mmap 里 vm_pgoff 是怎么来的？为什么能区分 BAR？**
答：用户态 `mmap(..., fd, offset)` 的 offset 会被内核按页对齐后放入 `vma->vm_pgoff`（单位是 PAGE_SIZE）。驱动约定 offset=0 → BAR0、offset=4096 → BAR2，把 mmap 的 offset 参数复用成"映射哪个 BAR"的选择器。用户态映射显存必须传 `1*PAGE_SIZE`，传 0 会映射到控制寄存器，这是本项目真实踩过的坑。

**Q5：`io_remap_pfn_range` 和 `remap_pfn_range` 区别？映射建立后内存内容变化用户态立刻可见吗？**
答：`io_remap_pfn_range` 用于设备 I/O 内存，映射属性按 device memory 配置（禁用投机访问、部分平台配置相应内存属性）；`remap_pfn_range` 用普通内存属性。二者都建立用户页表项指向 BAR 物理页，映射是共享的（页表指向同一物理地址），设备/CPU 写入后另一侧读到新值（配合必要的屏障）。BAR 是固定物理地址，不会被换出，所以可以安全映射。

**Q6：为什么 RISC-V 平台用 Legacy INTx 而不是 MSI-X？**
答：MSI-X 需要 PCI 设备和平台中断控制器支持以内存写事务投递中断。RISC-V virt 平台的 PLIC 不支持 MSI-X（这是项目真实踩过的坑），所以驱动用 `pci_alloc_irq_vectors(..., PCI_IRQ_LEGACY)` 退回传统电平触发中断。

**Q7：两个进程同时 ioctl dispatch 会怎样？怎么修？**
答：本项目 `kernel_done` 是设备级共享状态，两个进程并发 dispatch 会互相覆盖对方的寄存器参数和完成标志，A 的任务结果可能被 B 的唤醒消费，甚至两个 waiter 都醒或都睡。修复方向：a) 简单方案：设备级 mutex 串行化 ioctl；b) 正确方案：per-open 的私有上下文 + 设备级"任务队列"（提交队列/完成队列），真实 GPU 驱动就是这么做的。

**Q8：wait_event_interruptible 被信号打断会怎样？本项目代码有什么问题？**
答：信号打断时宏返回 `-ERESTARTSYS`（或 `-ERESTART_RESTARTBLOCK`），且**条件可能仍未满足**。本项目的 ioctl 直接忽略返回值返回 0，用户态会以为 kernel 执行完成，但设备可能还在跑或已完成而 kernel_done 未被消费。规范写法：检查返回值，返回 `-ERESTARTSYS` 让系统调用框架重启等待；或者返回 `-EINTR` 并保证状态一致性（比如任务已提交就继续等）。

**Q9：buddy 分配器为什么要 2 的幂？项目里引发了什么问题？**
答：2 的幂让"分裂"和"合并"都有确定规律（buddy 地址 = offset ^ block_size），且每档一条链表 O(1) 找到合适块。代价是内部碎片：请求向上取整。项目里 2.35GiB 权重整体申请会被 round 到 4GB 直接爆显存，4 块方案每块 683MB round 到 1GB 总占 3.95GiB 仍超，最终拆 8 个 region（每 4 层 228MB→round 256MB）总占 3.5GiB 才装下。

**Q10：gpuMemcpy 就是 memcpy，为什么可行？和真实 GPU 的 DMA 差在哪？**
答：因为驱动把 BAR2 显存 mmap 成了用户态普通地址空间，CPU 读写这段地址会经 PCI 总线到达设备内存，memcpy 即完成拷贝。真实 GPU 用 DMA 引擎：CPU 只写命令描述符，数据搬运由 DMA 控制器完成，不占 CPU、带宽高、可异步。mmap 直通适合小数据和控制路径，大数据必须 DMA。

**Q11：为什么显存分配放在用户态而不放驱动？**
答：分层原则：内核只提供机制（mmap 出整块显存 + dispatch），策略（怎么切分、碎片管理、缓存策略）放用户态，便于迭代调试（用户态崩溃可重跑，内核态崩溃整机死）、避免内核态复杂算法维护成本、不同框架（CUDA/Mesa）可以有自己的分配器。真实 GPU 驱动也是这么分层的。

**Q12：kernel 的参数怎么传给设备上跑的汇编？**
答：用户态把 args 结构体写到显存某块 buffer，ioctl 时把该 buffer 的显存偏移写进 KERNEL_ARGS 寄存器。设备启动 warp 时把该地址放进 a0 寄存器，汇编按固定偏移 lw 取各字段。C 结构体字段顺序和汇编 lw 顺序必须严格一致，错位不会 crash、只会算错，极难排查。

**Q13：什么是 warp、lane、divergence？min-PC 调度有什么问题？**
答：warp 是 GPU 的调度单位（本项目 32 线程），lane 是 warp 内单线程。同一 warp 的线程执行同一指令流，若 if/else 导致不同 lane 走不同分支就是 divergence。真实 GPU 有 reconvergence 栈让分叉的 lane 重新汇合；本项目用 min-PC 调度（每轮只执行 PC 最小的活跃 lane 集合）简化实现，缺陷是某个 lane 进入小 PC 的循环时会"饿死"其他大 PC 的 lane。

**Q14：softmax 那个 bug 的本质是什么？**
答：序列长度 S<32 时有 32-S 个空 lane，原代码让它们跳过初始化直接退出，但跨 lane 的树形归约无条件执行，把空 lane 寄存器里的脏值归约进来污染了 max/sum。修复：空 lane 也把累加器初始化为归约单位元（max 用 -INF、sum 用 0），参与归约但不影响结果。本质教训：**参与全局归约的每个成员，无论是否干活都必须处于合法状态**。

**Q15：浮点寄存器为什么要用"位模式"而不是 float？**
答：设备解释器的 FPR 是 uint32 数组，存 IEEE754 位模式。用 C float 直接存取会隐式发生类型转换/舍入语义混乱，所以边界处显式 `u32_to_float/float_to_u32` 转换。同理用户态传 scale 给设备时用 union 取 float 的位模式当 u32 传。这是软硬件接口处"数据就是位"的工程常识。

**Q16：一个 token 的推理要 launch 多少次 kernel？为什么慢？**
答：约 500 次（24 层 × 每层约 20 个算子 + 首尾）。每次 launch = 一次 ioctl + 100ms 定时中断等待，串行执行，所以极慢。优化方向：kernel fusion（合并算子减少 launch）、去掉 100ms 固定延时改为真正完成即中断、KV cache 走 device-side append 消除 48 次 D2H/H2D 往返。

**Q17：项目里你实际改了什么？（必问！）**
答：据实回答 6.3 节里你真正做的改进，每个用"问题 → 我的改法 → 验证"三句话讲。**绝不虚构没做的部分**；没做改进就回答"我完整复现并写了 XX 测试"，同时说清楚下一步想改什么、为什么。

**Q18：驱动有没有并发/同步问题？**
答：有。kernel_done 与寄存器组是设备级共享的，ioctl 未加锁，并发 dispatch 会互相覆盖；IRQF_SHARED 但 handler 不区分中断来源；remove 时若还有进程持 fd 在 ioctl 睡眠，释放资源会引发 UAF。真实驱动需要 per-file 上下文 + 设备锁 + remove 时的引用计数（或等待排空）。

**Q19：iowrite32 和直接 *(volatile u32*)p = x 有什么区别？**
答：`iowrite32`（及其底层 `writel`）不仅 volatile 访问，还带必要的内存屏障和平台特定的字节序/访问宽度处理，保证写顺序按程序序到达设备（写 DISPATCH 前所有参数寄存器必须已可见）。直接指针赋值在部分平台上可能被编译器重排或合并，且无屏障保证。

**Q20：从中学到的最重要的东西是什么？**
答（参考）："硬件是异步的，驱动是那个把所有异步对齐成同步的层——提交任务后睡眠、中断来唤醒，这个模式贯穿所有设备驱动。以及内存和寄存器本质都是'按地址放数据'，接口约定（寄存器布局、args 结构体、mmap offset 语义）一旦确立就是契约，改动任何一边都要有编译期或测试兜底。"

### 6.6 简历"技能清单"参考

> 结合本项目可诚实写入的技能行：

```text
● 熟悉 Linux 设备驱动开发：字符设备、PCI 设备驱动框架、中断/等待队列、
  mmap/ioctl 用户态接口、内核内存与并发基础；
● 熟悉 PCIe/PCI 设备原理：配置空间、BAR、MMIO、INTx/MSI 中断机制；
● 掌握用户态加速器编程模型：基于 mmap 的零拷贝访问、ioctl 任务提交、
  用户态内存分配器（buddy/slab）设计；
● 熟悉 RISC-V RV32 汇编与指令编码，理解 SIMT/warp/divergence 编程模型；
● 熟练使用 QEMU 虚拟化环境进行软硬件协同调试；掌握交叉编译、
  dmesg/gdb/bit-exact dump 对照等调试方法。
```

---

## 7. 附录

### 7.1 环境搭建实操步骤（已在本机全部搭好 ✅）

本机实际搭好的环境（2026-10-11 验证通过）：

| 组件 | 位置/版本 | 状态 |
| --- | --- | --- |
| 驱动交叉编译 | `riscv64-linux-gnu-gcc 11.4.0`（apt 安装） | ✅ `gpgpu_drv.ko` 已编出（566K） |
| kernel 汇编工具链 | `riscv64-unknown-elf-gcc 10.2.0`（apt 安装） | ✅ 21 个 `.bin` + `.dump` 已编出 |
| 内核头/构建树 | `~/kernel-devel/`（openEuler 官方 kernel-devel rpm，含 Module.symvers） | ✅ 驱动 `make driver` 可重复编译 |
| QEMU（含 gpgpu 设备） | `~/qemu-gpgpu/build/qemu-system-riscv64`（10.2.50 自编） | ✅ 编译通过 |
| RISC-V VM | `/home/cxr/vm/`：openEuler 24.03 LTS SP2 qcow2 + EDK2 固件 + 启动脚本 | ✅ 启动 + 部署 + 测试全链路跑通 |

**两个本机特有的坑（面试也可以讲，体现解决问题的能力）：**

1. **kernel-devel rpm 的预编译宿主工具是 RISC-V 二进制**（openEuler 在 riscv64 构建机上打包的）。x86 宿主机上 binfmt 拿 qemu 去执行，报
   `Could not open '/lib/ld-linux-riscv64-lp64d.so.1'`。解决：删掉 `scripts/{basic,mod,genksyms}/` 下预编译二进制，
   用宿主 gcc 从源码重建 `fixdep`/`modpost`/`mk_elfconfig`/`genksyms`。
2. **GCC 10 不支持 `rv32im_zicsr_zifencei_f` 写法**（GCC 12+ 才把 zicsr/zifencei 从 I 拆出来）。
   `kernels/Makefile` 已改为 `rv32imf`，编译结果等价。

```bash
# ① 驱动编译（已配好，直接可用）
cd ~/桌面/gpu_driver/gpgpu_driver && make driver && make test
# 产物: build/gpgpu_drv.ko + build/test_gpgpu

# ② GPU kernel 汇编编译
cd ~/桌面/gpu_driver/gpgpu_driver/kernels && make
# 产物: 每个 .S 对应 .bin（VRAM 中执行的指令）+ .dump（反汇编对照）

# ③ 启动 VM（1G 内存 + 2 vCPU + 128M VRAM，适配本机 3G 内存）
/home/cxr/vm/start_gpgpu_vm.sh   # SSH 转发在 localhost:12055

# ④ 纯 C 模块本机测试（不需要 QEMU！）
gcc -o /tmp/test_buddy lib/src/buddy.c lib/test/test_buddy.c -Ilib/include
/tmp/test_buddy

# 注意：完整 Qwen2-0.5B 推理需要 ~4GB VRAM + 2.35GB 权重文件，
# 本机只有 3G 内存，跑不动；学习阶段用 test_gpgpu 验证全链路即可。

# ⑤ 一键部署 + 验证（VM 启动后，宿主机执行）
/home/cxr/vm/deploy_and_test.sh
```

**2026-10-11 全链路验证结果（真实跑通，面试可直接引用）：**

```
== 2. PCI 枚举 ==
GPGPU 设备: 0000:00:01.0                      ← QEMU 虚拟设备挂在 PCI 总线上
== 3. 加载驱动 ==
insmod OK
crw------- 1 root root 234, 0 /dev/gpgpu       ← 字符设备节点自动创建
== 4. probe 日志 ==
gpgpu 0000:00:01.0: GPGPU device found! vendor=0x1234 device=0x1337
gpgpu 0000:00:01.0: GPGPU device id is 0x47505055   ← ASCII "GPPU"
gpgpu 0000:00:01.0: VRAM size: 128 MB
== 5. test_gpgpu ==
read back: 0x12345678                        ← mmap 写 VRAM 再读回，一致
ioctl took 109 ms                            ← dispatch → 等中断 → 返回
read back: 0x00000001                        ← GLOBAL_STATUS=1，内核执行完成
== 6. 中断计数 ==
17: 1 0  SiFive PLIC 33 Edge  gpgpu-kernel   ← 1 次 dispatch 正好 1 次中断
```

读日志的小技巧：`0x47505055` 是 `GPPU` 四个字符的 ASCII——设备 ID 寄存器写的是字符串常量，
驱动只是把它当整数读出来。面试官若问"为什么是 0x47505055"，这就是答案。

另外两条日志也值得注意：`loading out-of-tree module taints kernel` 和
`module verification failed ... tainting kernel`——guest 内核开了
`CONFIG_MODULE_SIG` 但没有 `SIG_FORCE`，未签名模块可以加载只是打 taint 标记。
```

### 7.2 关键命令速查（学习全程用）

```bash
# 驱动编译/部署（在 VM 内）
make driver && scp -P 12055 build/gpgpu_drv.ko root@localhost:/root/
insmod /root/gpgpu_drv.ko        # 加载，观察 dmesg
dmesg | grep -i gpgpu            # 看 probe 日志
cat /proc/devices | grep gpgpu   # 看设备号
ls -l /dev/gpgpu                 # 看设备节点
cat /proc/interrupts | grep gpgpu# 看中断计数
rmmod gpgpu_drv                  # 卸载，验证 remove 清理

# kernel 汇编编译/检查
cd kernels && make && head -50 gemm.dump   # 反汇编对照

# 用户态测试（VM 内）
./test_transformer ./weights /tmp          # 端到端 sanity

# gdb 调试用户态程序（host 或 VM）
gdb ./test_buddy
(gdb) break buddy_alloc
(gdb) run
(gdb) x/21gx alloc->free_list              # 看 21 条空闲链表头

# 权重一致性校验（README 的 checksum gate）
md5sum weights/*.bin
ssh -p 12055 root@localhost 'cd /root && md5sum weights/*.bin'
```

### 7.3 术语速查表（面试前过一遍）

| 术语 | 一句话解释 |
|---|---|
| PCI 配置空间 | 每个 PCI 设备自带的 256B+ 元信息区（vendor/device ID、BAR 等），系统枚举时读它 |
| BAR | 设备向系统申请的内存/IO 地址窗口，数量与大小由设备决定 |
| MMIO | 内存映射 IO：把设备寄存器映射到内存地址空间，用读写指令访问 |
| INTx / MSI-X | 传统共享中断线 / 基于消息写入的中断，后者更快更灵活 |
| `__iomem` | 内核标记：设备内存映射后的指针，禁止直接解引用 |
| 中断上下文 | 中断 handler 运行的环境：无进程上下文、不可睡眠 |
| 等待队列 | 内核睡眠/唤醒机制：进程入睡等条件成立，别人 wake_up 叫醒 |
| `copy_from_user` | 用户→内核拷贝，带地址合法性检查；用户指针禁止直接解引用 |
| 伙伴系统 buddy | 按 2 的幂分块、分裂/合并的内存管理算法 |
| slab | 面向小对象的缓存机制：整页切成固定 slot 用链表管 |
| 外部/内部碎片 | 总空间够但无连续大块 / 分配块内部浪费的空间 |
| SIMT | 单指令多线程：一个 warp 执行同一指令流，lane 按 ID 分工 |
| warp / lane | GPU 调度单位（本项目 32 线程）/ warp 内单线程 |
| mhartid | RISC-V 硬件线程号，本项目用它编码 block/warp/lane 身份 |
| shuffle | warp 内线程交换寄存器数据的操作 |
| divergence | warp 内线程走不同分支，真实 GPU 有 reconvergence 机制 |
| KV cache | 缓存历史 token 的 K/V 矩阵避免重算的机制 |
| bit-exact | 逐位一致：浮点结果的位模式完全相同 |
| ULP | 浮点误差单位（最后一位精度），衡量数值误差大小 |
| D2H / H2D | Device→Host / Host→Device 的数据传输方向 |

### 7.4 参考资料

- 项目源码：`/home/cxr/桌面/gpu_driver/gpgpu_driver`（本机）
- 配套 QEMU 仓库：https://github.com/gevico/qemu-camp-2026-exper-Eclipse-Arrebol
- 宋宝华《Linux 设备驱动开发详解》（驱动入门）
- 《Linux 内核设计与实现》(LKD) 第 4/7/11/12 章（等待队列/中断/内存管理）
- 《RISC-V 开放架构设计之道》（汇编基础，前 3 章够用）
- 内核源码 `Documentation/PCI/pci.rst`（PCI 驱动 API）
- CUDA Programming Guide 前 2 章（SIMT 概念补充）

---

*文档完。下一步行动：按第 4 章阶段 1 开始，第 0 周先装工具链、跑 `make list`、读一遍 git log。*
