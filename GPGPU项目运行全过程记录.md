# GPGPU 项目运行全过程记录

> 记录日期：2026-10-11 ｜ 状态：**全链路验证通过**
> 本文档记录从零搭建环境到完整跑通 gpgpu-driver 项目的全过程，含真实命令与真实输出，可作面试复述底稿。

---

## 0. 这个项目是什么、什么算"跑通"

项目是一个**教学用虚拟 GPGPU（GPU 里的通用计算部分）**，整体分为四层：

| 层 | 组件 | 跑在哪 |
|---|---|---|
| 硬件层 | QEMU 里新加的 `gpgpu` PCI 虚拟设备（含 128MB VRAM、能执行 RV32 指令、能发中断） | 宿主机 |
| 内核层 | Linux 字符设备驱动 `gpgpu_drv.ko`（PCI probe、mmap 显存、ioctl 派发、中断处理） | guest（openEuler RISC-V VM） |
| 用户层 | 运行库 libgpgpu + 测试程序 test_gpgpu | guest |
| "核函数"层 | 21 个 RV32 汇编 kernel（.S → .bin），由设备加载执行 | 由 guest 经驱动上传到设备 |

**"跑通"的定义**（也是本次验证的目标）：

> 编译全部产物 → VM 启动 → 驱动加载 → `/dev/gpgpu` 出现 → 用户程序 mmap 显存读写成功 → ioctl 派发后设备执行 kernel 并产生硬件中断 → 驱动唤醒等待队列 → 程序读到完成标志 → `/proc/interrupts` 中断计数 +1

---

## 1. 环境清单（装了哪些东西）

| 项目 | 内容 |
|---|---|
| 宿主机 | Ubuntu（kernel 6.8.0-138-generic），内存约 3GB |
| 驱动交叉编译器 | `riscv64-linux-gnu-gcc` 11.4.0（编 .ko 和 guest 用户态程序） |
| GPU kernel 编译器 | `riscv64-unknown-elf-gcc` 10.2.0（编 RV32 裸机 .bin） |
| 内核构建树 | openEuler kernel-devel rpm 解压到 `~/kernel-devel/usr/src/kernels/6.6.0-98.0.0.103.oe2403sp2.riscv64`（含 Module.symvers；宿主工具已用宿主 gcc 重编） |
| 定制 QEMU | `~/qemu-gpgpu/`，含 `hw/gpgpu` 虚拟设备，QEMU 10.2.50，以 `-O2` 重建 |
| 客户机 | `~/vm/`：openEuler 24.03 LTS SP2 riscv64（qcow2 + EDK2 固件），1G 内存 / 2 vCPU / 128M VRAM，SSH 宿主机 12055 → guest 22 |
| 脚本 | `~/vm/start_gpgpu_vm.sh`（启动）、`~/vm/deploy_and_test.sh`（一键部署验证） |

---

## 2. 编译三个环节

### 2.1 驱动 + 测试程序（riscv64-linux-gnu 交叉编译）

```bash
cd /home/cxr/桌面/gpu_driver/gpgpu_driver
make driver   # 外部模块编译: make -C $(KDIR) M=$(PWD) ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- modules
make test     # riscv64-linux-gnu-gcc -o build/test_gpgpu test/test_gpgpu.c
```

产物：`build/gpgpu_drv.ko`、`build/test_gpgpu`。Makefile 的 `KDIR` 已指向上面的 kernel-devel 构建树（guest 内核同版本，符号一致所以 insmod 不会报 vermagic 错）。

### 2.2 GPU kernels（riscv64-unknown-elf 交叉编译）

```bash
cd /home/cxr/桌面/gpu_driver/gpgpu_driver/kernels
make   # 自动发现 *.S → *.bin（还附带 *.dump 反汇编）
```

产物：21 个 `.bin`（如 `argmax.bin`、`broadcast_add.bin`）。编译参数 `-march=rv32imf -mabi=ilp32f -nostdlib -Wl,-Ttext=0x0`——裸机代码，从地址 0 开始，由设备直接加载执行。

### 2.3 QEMU（带 gpgpu 设备）

```bash
cd /home/cxr/qemu-gpgpu
mkdir -p build && cd build
../configure --extra-cflags='-O2' --target-list=riscv64-softmmu   # 只编 riscv64 软模拟目标
make -j$(nproc)
```

产物：`build/qemu-system-riscv64`。注意 `--extra-cflags='-O2'` 是必须的——camp 的默认参数是 `-O0 -g3`，TCG 纯软件模拟下会慢 5~10 倍，guest 启动都要半小时。

---

## 3. 启动 VM

```bash
/home/cxr/vm/start_gpgpu_vm.sh
```

该脚本等价于：

```bash
/home/cxr/qemu-gpgpu/build/qemu-system-riscv64 \
  -nographic -machine virt,pflash0=pflash0,pflash1=pflash1,acpi=off \
  -smp 2 -m 1G \
  -blockdev node-name=pflash0,driver=file,read-only=on,filename=RISCV_VIRT_CODE.fd \
  -blockdev node-name=pflash1,driver=file,filename=RISCV_VIRT_VARS.fd \
  -drive file=openEuler-24.03-LTS-SP2-riscv64.qcow2,format=qcow2,id=hd0,if=none \
  -device virtio-blk-device,drive=hd0 \
  -device virtio-net-device,netdev=usernet \
  -netdev user,id=usernet,hostfwd=tcp::12055-:22 \
  -device gpgpu,vram_size=134217728
```

关键点：

- **EDK2 UEFI 固件**（两个 pflash）引导 riscv64 的 openEuler；
- **`-device gpgpu,vram_size=134217728`** 把虚拟 GPU 挂到 PCI 总线上，128MB 显存。`vram_size` 必须写**原始整数**——QEMU 的 `DEFINE_PROP_UINT64` 不认 `"128M"` 这种带后缀的字符串；
- `hostfwd` 把宿主机的 12055 端口转发进 guest 的 sshd（22 端口），宿主和 guest 之间全靠它通信。

启动后等 guest 引导完成、sshd 就绪，用 `ssh -p 12055 root@localhost` 免密登录（已推过公钥；首登密码（本地 VM 密码，已省略））。

---

## 4. 部署 + 6 步验证（真实输出）

```bash
/home/cxr/vm/deploy_and_test.sh
```

脚本做的事：scp 上传 .ko / test_gpgpu / kernels 的 .bin → 检查 PCI 枚举 → insmod → 看 probe 日志 → 跑测试 → 查中断计数。

### 真实输出与解读

```
== 1. 上传文件 ==
（.ko、test_gpgpu → /root，*.bin → /tmp）

== 2. PCI 枚举 ==
GPGPU 设备: 0000:00:01.0
```

**证明了什么**：QEMU 的虚拟设备真的挂上了 PCI 总线（BDF 为 00:01.0），vendor 0x1234 匹配。这是后面一切的前提——驱动 probe 靠的就是 PCI 枚举。

```
== 3. 加载驱动 ==
insmod OK
crw------- 1 root root 234, 0 /dev/gpgpu
```

**证明了什么**：模块加载成功（无 vermagic/签名错误），probe 跑完后 `device_create` 自动创建了字符设备节点，主设备号 234、次设备号 0。

```
== 4. probe 日志 ==
gpgpu 0000:00:01.0: GPGPU device found! vendor=0x1234 device=0x1337
gpgpu 0000:00:01.0: GPGPU device id is 0x47505055
gpgpu 0000:00:01.0: VRAM size: 128 MB
```

**证明了什么**：驱动的寄存器 ABI 和设备端对上了——`0x47505055` 正是 ASCII **"GPPU"**（设备 ID 寄存器里存的是字符串常量，面试常问）。VRAM 大小也从寄存器读出，与启动参数一致。到这里说明 BAR 映射、寄存器读写全部正确。

```
== 5. test_gpgpu ==
read back: 0x12345678
ioctl took 109 ms
read back: 0x00000001
```

**证明了什么**：

- 第一条：用户态 `mmap` 映射 BAR2 显存，写入 0x12345678 再读回一致——**mmap 显存通路 OK**；
- 第二条：`ioctl(GPGPU_IOC_DISPATCH)` 把 kernel 地址/网格/块/共享内存参数写入设备寄存器，写 DISPATCH 寄存器触发执行，然后 `wait_event_interruptible` 睡下——109ms 后醒来，说明**设备执行 + 硬件中断 + 驱动唤醒等待队列整条链 OK**；
- 第三条：读回 GLOBAL_STATUS=1——**RV32 kernel 在设备上真的跑完并写了结果**。

```
== 6. 中断计数 ==
17: 1 0  SiFive PLIC 33 Edge  gpgpu-kernel
```

**证明了什么**：PLIC（RISC-V 平台中断控制器）上 33 号中断属于 gpgpu 设备，1 次 dispatch 恰好 +1。说明驱动注册的 IRQ 号与 QEMU 设备端的中断连线完全一致，没有多发也没有漏发。

**结论**：虚拟设备 ↔ 驱动寄存器 ABI ↔ 中断投递 ↔ 用户态 mmap/ioctl，四个层全部打通。

---

## 5. 踩过的坑（面试素材，每个都能讲一段故事）

1. **kernel-devel rpm 里的"宿主工具"是 RISC-V ELF**。openEuler 官方 rpm 是在 riscv64 构建机上打的包，里面预编译的 fixdep/modpost/mk_elfconfig/genksyms 是目标架构二进制。x86 宿主机跑外部模块编译时 binfmt 报缺 `/lib/ld-linux-riscv64-lp64d.so.1`。**修法**：删掉预编译二进制，用宿主 gcc 从源码重编这几个小工具。本质：交叉编译外部模块时，`make -C $(KDIR) M=...` 里跑的脚本/工具必须用宿主工具链。

2. **GCC 版本与 RISC-V ISA 字符串演进**。GCC 12+ 把 `zicsr`/`zifencei` 从 I 扩展里拆出来单独写（`rv32im_zicsr_zifencei_f`），而本机的 `riscv64-unknown-elf-gcc` 是 10.2.0，只认老写法 `rv32imf`。kernels/Makefile 已改成 `rv32imf` 并留了注释。

3. **QEMU TCG 模拟性能**。camp 默认 `--extra-cflags='-O0 -g3'` 是给调试用的，纯软件模拟（TCG）下慢 5~10 倍，guest 启动要半小时以上。用 `-O2` 重建 QEMU 本体即可，不影响调试能力（gdb 挂在 guest 内核上，不依赖 QEMU 的 -O0）。

4. **QEMU 属性解析**：`DEFINE_PROP_UINT64` 类型属性只接受原始整数，`vram_size=128M` 这种带后缀写法会解析失败，必须写 `vram_size=134217728`。

5. **宿主内存管理**：宿主机只有 3GB 内存，VSCode 占 2GB，QEMU 再开 1G 就贴线了；并行跑重任务会让 swap 打满、整机抖动。教训：给 guest 只开 1G 内存、重任务串行跑。完整 Qwen2-0.5B 推理（需 ~4GB VRAM + 2.35GB 权重）本机跑不动，学习阶段用 test_gpgpu 验证即可。

6. **insmod 后的 taint 日志**：guest 内核开了 `CONFIG_MODULE_SIG` 但没开 `SIG_FORCE`，所以未签名模块可以加载，只是内核打一条 `module verification failed: signature and/or required key missing - tainting kernel`。这是安全特性而不是错误，能讲清两者区别是加分项。

7. **SELinux 全程 enforcing**：整条链路在 SELinux 开启状态下跑通，没有任何 AVC 拒绝，从头到尾不需要关 SELinux（`setenforce 0` 从未使用）。

---

## 6. 日常复现（两条命令）

```bash
# ① 启动 VM（一条命令，nohup 后可关终端）
/home/cxr/vm/start_gpgpu_vm.sh

# ② 一键部署 + 6 步验证
/home/cxr/vm/deploy_and_test.sh
```

改完驱动后重编再跑 ② 即可（deploy 脚本会 scp 最新的 .ko 和 kernels）：

```bash
cd /home/cxr/桌面/gpu_driver/gpgpu_driver
make driver && /home/cxr/vm/deploy_and_test.sh
```

手动登录 VM：`ssh -p 12055 root@localhost`（免密；首登密码（本地 VM 密码，已省略））。

---

## 7. 关键文件路径速查

| 用途 | 路径 |
|---|---|
| 驱动源码 | `/home/cxr/桌面/gpu_driver/gpgpu_driver/gpgpu_drv.c` |
| 驱动/测试 Makefile | `/home/cxr/桌面/gpu_driver/gpgpu_driver/Makefile` |
| GPU kernels（.S/.bin） | `/home/cxr/桌面/gpu_driver/gpgpu_driver/kernels/` |
| 用户态运行库 | `/home/cxr/桌面/gpu_driver/gpgpu_driver/lib/` |
| 内核构建树 | `/home/cxr/kernel-devel/usr/src/kernels/6.6.0-98.0.0.103.oe2403sp2.riscv64/` |
| 定制 QEMU | `/home/cxr/qemu-gpgpu/`（二进制 `build/qemu-system-riscv64`） |
| VM 镜像与脚本 | `/home/cxr/vm/`（start_gpgpu_vm.sh、deploy_and_test.sh） |
| VM 引导日志 | `/home/cxr/vm/boot.log` |
| 学习计划主文档 | `/home/cxr/桌面/gpu_driver/GPGPU项目学习计划.md`（§7.1 环境搭建章节与本记录对应） |
