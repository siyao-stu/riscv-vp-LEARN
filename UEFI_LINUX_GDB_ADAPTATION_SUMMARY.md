# GUI-VP Kit 中 UEFI + Linux 启动与 GDB 适配总结

## 1. 目标与最终结果

本轮适配的目标不是单点修 bug，而是把 GUI-VP Kit 的 RV64 多核平台打通为一条可调试、可复现、可维护的完整启动链路：

`OpenSBI -> UEFI(EDK2) -> Linux EFI Stub -> Linux Kernel -> initramfs/init -> userspace`

同时，调试链路也要完整覆盖：

`OpenSBI symbols + UEFI symbols + Linux kernel symbols + VP GDB remote debugging`

最终达成的效果包括：

- UEFI 可以从 virtio-blk 提供的磁盘镜像中识别 ESP 并加载 `BOOTRISCV64.EFI`
- `BOOTRISCV64.EFI` 改为直接使用 Linux 的 EFI `Image`，成功进入内核
- 内核早期阶段可见日志，可在 `_start_kernel`、`start_kernel` 等位置下断点
- 内核 GDB 已具备行号信息，可做源码级调试
- 同一个 GDB 会话中可以同时加载 OpenSBI、UEFI 和 Linux 内核符号
- 内核可通过 initrd/initramfs 启动到 `/sbin/init`
- VP 在 GDB 单步调试时不再因未处理异常直接崩溃退出

## 2. 整体启动与调试链路

### 2.1 启动链路

当前采用的启动方式如下：

1. VP 启动 OpenSBI
2. OpenSBI 跳转进入 EDK2/UEFI
3. UEFI 在 virtio-blk 暴露的磁盘上扫描 GPT/ESP
4. UEFI 从 ESP 加载 `EFI/BOOT/BOOTRISCV64.EFI`
5. `BOOTRISCV64.EFI` 实际上是 Linux 内核的 EFI `Image`
6. Linux EFI stub 完成 EFI 环境下的参数、FDT、initrd 处理，并执行 `ExitBootServices`
7. 内核进入 RISC-V 汇编入口和 `start_kernel`
8. 内核挂载 initrd/initramfs，并执行 `/sbin/init`

### 2.2 调试链路

调试链路分成三个层次：

1. OpenSBI：使用 `fw_jump.elf` 符号
2. UEFI：通过 `tools/riscv_uefi_symbols.py` 动态加载 UEFI 模块符号
3. Linux Kernel：通过 `vmlinux` + 固定虚拟基址 `0xffffffff80000000` 加载内核符号

这样可以在同一个 GDB 会话里跨阶段调试，不需要在 OpenSBI、UEFI、Linux 三个阶段之间反复切换调试配置。

## 3. 主要适配点

### 3.1 virtio-blk 与可启动磁盘镜像

为了让 UEFI 能像真实虚拟平台那样从磁盘加载 EFI 程序，需要先在 VP 中打通 virtio-blk MMIO 设备，并提供可识别的磁盘镜像。

关键适配点：

- 在 VP 中实现/修正 virtio-blk MMIO 设备路径，使其能被 UEFI 正确识别
- 修正 DMA/TLM 返回状态，避免 UEFI 访问磁盘时出现隐式失败
- 构造带 GPT 分区表的磁盘镜像，并创建 ESP 分区
- 将 `BOOTRISCV64.EFI`、`startup.nsh`、`initrd.cpio` 等文件放入 ESP

原理：

- UEFI 不关心 Linux 的 buildroot 产物如何生成，它只关心是否存在一个可枚举的块设备，以及块设备上是否有合法的 EFI System Partition
- 只有当块设备模型、分区表、文件系统三者同时正确时，UEFI Shell 中才会出现类似 `FS0:` 的文件系统映射

### 3.2 使用 Linux EFI `Image` 作为 `BOOTRISCV64.EFI`

这里的一个关键结论是：`BOOTRISCV64.EFI` 不能继续使用简单 `objcopy` 出来的裸内核镜像，而应直接使用内核构建产出的 EFI `Image`。

原因：

- UEFI 的 `LoadImage`/`StartImage` 期望加载的是合法 PE/COFF EFI 映像
- Linux 的 EFI stub 已经把内核包装成 UEFI 可执行映像
- 直接把 EFI `Image` 放到 `EFI/BOOT/BOOTRISCV64.EFI`，才能走标准的 UEFI 启动路径

这一步是把“UEFI 能加载一个文件”变成“UEFI 能真正按标准方式启动 Linux”的核心适配点。

### 3.3 UEFI 启动流程诊断

在适配过程中，最初的现象是 UEFI 看起来卡在 `StartImage()` 附近，无法判断问题到底在 UEFI 侧还是 Linux 侧。

对此做了定点诊断：

- 在 `edk2/MdeModulePkg/Library/UefiBootManagerLib/BmBoot.c` 的 `EfiBootManagerBoot()` 周围增加日志
- 重点打印 `LoadImage()` 返回值、`StartImage()` 调用前后状态、镜像句柄、错误路径信息
- 同时观察 `TimerDriverSetTimerPeriod(0x0)` 等日志

得到的结论是：

- `LoadImage()` 成功
- `StartImage()` 已经进入内核 EFI stub
- `TimerDriverSetTimerPeriod(0x0)` 更符合 `ExitBootServices` 发生时的正常行为，而不是 UEFI 死锁

也就是说，问题控制路径已经从 UEFI 转移到 Linux EFI stub / kernel early boot。

### 3.4 Linux EFI stub 早期定位

为了确认内核到底走到了哪里，对 EFI stub 增加了早期日志：

- `drivers/firmware/efi/libstub/efi-stub-entry.c`
- `drivers/firmware/efi/libstub/fdt.c`

重点观察的内容：

- EFI stub 是否真正进入
- FDT 是否成功准备
- initrd 参数是否被识别
- `ExitBootServices` 前后流程是否执行完成

原理：

- EFI stub 是 UEFI 世界和 Linux 世界之间的交界层
- 如果 UEFI 已执行 `StartImage()`，但内核没有正常启动，最有价值的定位点通常不是通用内核初始化代码，而是 EFI stub 中处理 FDT、命令行、initrd、退出 Boot Services 的那段逻辑

### 3.5 RISC-V 内核最早期入口可见化

仅靠 EFI stub 日志还不足以排除“进入内核后马上死掉”的情况，因此在：

- `arch/riscv/kernel/head.S`

中给 `_start_kernel` 增加了极早期 SBI 控制台标记输出，例如 `K\n`。

原理：

- 这类输出发生在常规 console 子系统初始化之前
- 如果能看到该标记，说明控制权已经从 EFI stub 成功转移到内核最早期入口
- 这比等待常规 `printk` 更早，也更适合定位“刚进内核就停”的问题

### 3.6 内核源码级 GDB 调试能力

一开始虽然能给符号下断点，但缺少行号信息，导致源码级调试体验不足。

为此在内核配置中启用了：

- `CONFIG_DEBUG_INFO_DWARF4=y`

并关闭了仅无调试信息的配置路径。

同时确认当前内核：

- 未开启 `CONFIG_RELOCATABLE`
- 未开启 `CONFIG_RANDOMIZE_BASE`

因此可以安全地将 `vmlinux` 固定加载到：

- `0xffffffff80000000`

原理：

- `vmlinux` 提供的是完整 ELF 和 DWARF 调试信息
- `Image` 用于启动，但不适合作为源码级调试主符号文件
- 当内核不开启可重定位/KASLR 时，GDB 的 `add-symbol-file vmlinux 0xffffffff80000000` 是稳定有效的

### 3.7 OpenSBI + UEFI + Linux 三段符号统一加载

在 Makefile 中改造了 GDB 连接目标，使一个 GDB 会话就能覆盖三个阶段。

典型做法包括：

- `symbol-file` 加载 OpenSBI 的 `fw_jump.elf`
- 通过 `tools/riscv_uefi_symbols.py` 加载 UEFI 模块符号
- 通过 `add-symbol-file` 加载 Linux `vmlinux`

这样调试时可以连续观察：

- OpenSBI 跳转 UEFI
- UEFI 调用 `StartImage`
- EFI stub 接管
- 内核进入 `_start_kernel` / `start_kernel`

这比为不同阶段分别维护多个 GDB 配置更高效，也更适合复杂启动路径的联调。

### 3.8 串口输出恢复

在内核开始执行后，一度出现“GDB 里能看到代码继续，但串口没有日志”的情况。最终确认问题并不在 EFI，而在内核串口驱动配置。

启用的关键配置包括：

- `CONFIG_SERIAL_8250=y`
- `CONFIG_SERIAL_8250_CONSOLE=y`
- `CONFIG_SERIAL_8250_NR_UARTS=4`
- `CONFIG_SERIAL_8250_RUNTIME_UARTS=4`

并使用如下命令行思路：

- `console=ttyS0,115200n8`
- `earlycon=uart8250,mmio,0x10000000,115200`

原理：

- DT 中串口是 `ns16550a@0x10000000`
- `earlycon` 提供正式 console 初始化前的早期输出
- `console=ttyS0,...` 负责正式 console 接管后的持续输出

### 3.9 initrd / initramfs 启动路径修正

启动 Linux 后，曾遇到根文件系统相关 panic，以及 `/sbin/init` 无法正确拉起的问题。

主要调整有两部分。

第一部分是命令行修正：

- 从 `root=/dev/ram0 rw` 调整为 `rdinit=/sbin/init`

原因：

- 当前链路本质上是通过 EFI 传入 initrd/initramfs，让内核直接从内存中的 rootfs 启动
- 这里并不适合再把它当成传统块设备根文件系统来挂载

第二部分是内核配置与 initrd 生成方式修正：

- 启用 `CONFIG_BLK_DEV_INITRD=y`
- initrd 生成从简单 `cpio` 打包切换到内核自己的 `usr/gen_initramfs.sh`
- 在生成过程中显式注入必要的设备节点：
  - `/dev/console`
  - `/dev/null`

原理：

- 若 `CONFIG_BLK_DEV_INITRD` 未开启，内核即使收到了 initrd 相关参数，也无法按预期使用
- `/dev/console` 缺失时，`/sbin/init` 可能已经被执行，但控制台无法正常建立，表象上会像“系统卡死”或“init 没起来”
- 使用内核自带 `gen_initramfs.sh` 更接近内核自身对 initramfs 格式的预期，稳定性更高

### 3.10 VP 侧 GDB 单步稳定性修复

在内核已经能启动并被 GDB 调试后，又暴露出一个新的问题：

- 使用 `n` 或某些单步动作时，VP/SystemC GDB server 会直接抛出异常并退出
- 典型报错是 `E549 unknown exception`

为此做了两类修复。

第一类是运行参数修正：

- 在 GDB 运行目标中加入 `--debug-cont-sim-on-wait`

其作用是改善 VP 在等待 GDB 控制、继续仿真、进入断点等场景下的协作方式，减少单步时的仿真控制异常。

第二类是 GDB server 异常处理增强：

- 修改 `riscv-vp-plusplus/vp/src/core/common/gdb-mc/gdb_server.cpp`
- 对 `sc_core::sc_unwind_exception` 单独处理并继续向外抛出
- 对其他异常增加更细的日志，打印 `cmd`、原始 `pkt`、异常类型
- 出错时返回 GDB 协议错误包，而不是直接把 SystemC 进程打死

原理：

- `sc_unwind_exception` 是 SystemC 控制流的一部分，不能粗暴吞掉
- 其他异常若全部落入不透明的 `catch (...)`，只会看到 “unknown exception”，无法继续定位
- GDB server 对异常更稳健后，即使单条命令失败，也不会拖垮整个 VP 调试会话

## 4. 关键问题与解决过程

### 问题 1：UEFI 看起来卡在 `StartImage()`

现象：

- UEFI 能看到启动项，但启动 Linux 时像是停在 `StartImage()` 附近

定位方法：

- 给 `BmBoot.c` 增加 `LoadImage()` / `StartImage()` 前后日志
- 结合 UEFI Timer 相关日志判断流程位置

结论与解决：

- UEFI 本身并未卡死，而是已经把控制权交给 Linux EFI stub
- 后续问题应转到内核 EFI stub 和 early boot 继续定位

### 问题 2：内核是否真正进入了入口无法确认

现象：

- UEFI 返回路径不明确
- 常规串口日志又没有及时出现

定位方法：

- 给 EFI stub 增加日志
- 在 `head.S` 中添加最早期 SBI 标记输出

结论与解决：

- 可以明确看到控制流已进入 `_start_kernel`
- 成功把问题边界从 UEFI 缩小到内核初始化早期

### 问题 3：GDB 能下断点但没有源码行号

现象：

- 只能按符号名断点，源码单步体验差

解决：

- 启用 `CONFIG_DEBUG_INFO_DWARF4=y`
- 使用 `vmlinux` 作为调试符号文件

结果：

- `start_kernel`、`setup_arch`、`mm_init` 等可做行级调试

### 问题 4：内核 panic，根文件系统路径不对

现象：

- 使用 `root=/dev/ram0` 时，根文件系统路径与 EFI 加载 initrd 的实际模式不匹配

解决：

- 改为 `rdinit=/sbin/init`
- 明确采用 initrd/initramfs 直启模型

结果：

- 启动路径与系统结构一致，不再误把当前方案当成块设备根启动

### 问题 5：`/sbin/init` 启动不稳定或无控制台

现象：

- 即使到达用户态附近，仍可能没有正常交互输出

解决：

- 启用 `CONFIG_BLK_DEV_INITRD`
- 用 `usr/gen_initramfs.sh` 生成 initrd
- 注入 `/dev/console` 和 `/dev/null`

结果：

- 内核可成功运行 `/sbin/init`
- 用户态控制台建立正常

### 问题 6：串口日志缺失

现象：

- 内核继续执行，但串口输出缺失，影响判断

解决：

- 启用 8250 串口和 console 支持
- 配合 `earlycon` 与 `console=ttyS0`

结果：

- 早期和正式串口输出都恢复，可持续观察内核启动日志

### 问题 7：GDB 单步导致 VP 崩溃

现象：

- `n`、单步、部分断点场景下，VP 直接异常退出

解决：

- 增加 `--debug-cont-sim-on-wait`
- 改造 `gdb_server.cpp` 的异常处理逻辑

结果：

- VP 不再因单步命令的异常直接崩溃退出

### 问题 8：单步不再崩溃，但仍有 `SimulationTrap` 噪声

现象：

- VP 日志中反复出现：GDB `cmd=m` 内存读取触发 `SimulationTrap`

当前结论：

- 这说明问题已从“VP 被异常打死”收敛为“某些 GDB memory read 包访问了 trap 地址”
- 也就是说，系统稳定性问题已显著收敛，剩余工作主要是把该类地址访问变成可预期的调试错误返回，而不是噪声日志

这属于后续可继续优化的调试体验问题，不再是启动链路阻塞问题。

## 5. Makefile 与工作流改造要点

本轮还对调试工作流做了系统化改造，核心目标是减少重复手工操作。

主要改造包括：

- 增加内核 GDB 连接目标
- 增加自动化内核早期断点目标
- 在原有 UEFI GDB 目标中合并 Linux 内核符号加载
- 引入 `KERNEL_VIRT_BASE ?= 0xffffffff80000000`
- 在 VP GDB 运行目标中加入 `--debug-cont-sim-on-wait`
- 在 EFI SD 构建目标中自动生成带设备节点的 `initrd.cpio`

这些改造带来的直接收益是：

- 启动与调试步骤更可复现
- OpenSBI/UEFI/Linux 联调不再依赖临时人工命令
- 新问题出现时，更容易快速回到同一调试基线

## 6. 适配背后的核心原理

如果把本次适配抽象成方法论，核心有四点。

### 6.1 先打通标准启动路径，再做局部修补

例如：

- 不再用临时裸镜像顶替 EFI 程序，而是直接让 UEFI 加载 Linux EFI `Image`
- 不再把 initrd 启动误当作块设备根启动

只有路径本身符合 UEFI 与 Linux 的标准模型，后续定位才稳定。

### 6.2 诊断要落在控制权交接边界上

这次最关键的几个观察点，都是边界位置：

- `LoadImage` / `StartImage`
- EFI stub 入口
- `_start_kernel`
- `start_kernel`
- `/sbin/init`

边界点最能快速回答“控制权到底到了哪里”，比在大范围代码里盲搜更有效。

### 6.3 调试信息、日志、符号表必须同时具备

单独有其中之一都不够：

- 没有串口日志，很难知道系统是否继续运行
- 没有 DWARF，很难做行级调试
- 没有统一符号加载，很难跨 OpenSBI/UEFI/Kernel 联调

三者一起完善后，定位效率会明显提升。

### 6.4 先把“崩溃”收敛成“可解释错误”

VP GDB 的问题就是典型例子：

- 第一阶段目标不是立即消灭所有异常
- 而是先避免 VP 因异常直接退出
- 再把“unknown exception”变成带 `cmd`、`pkt`、异常类型的可定位信息

只有先做到这一点，后续才能继续修精细问题，例如 `SimulationTrap` 的 read-memory 处理。

## 7. 当前状态与后续建议

### 7.1 当前状态

当前这条链路已经具备以下能力：

- 从 OpenSBI 启动进入 UEFI
- UEFI 从 virtio-blk 的 ESP 加载 Linux EFI `Image`
- Linux 通过 EFI stub 完成启动交接
- 内核具备早期日志与源码级 GDB 调试能力
- initrd/initramfs 路径可启动到 `/sbin/init`
- 同一 GDB 会话可联调 OpenSBI、UEFI、Linux
- VP 在单步时不再直接崩溃

### 7.2 后续建议

后续若继续完善，建议优先做两项：

1. 在 VP 的 GDB memory-read 路径中，把 `SimulationTrap` 转换成更温和的协议级错误返回
2. 将当前 UEFI + Linux + GDB 联调方法补充进项目主 README 或单独维护为开发者调试指南

## 8. 涉及的关键文件

本轮适配与调试过程中重点涉及的文件包括：

- `Makefile`
- `edk2/MdeModulePkg/Library/UefiBootManagerLib/BmBoot.c`
- `buildroot_rv64/linux_rv64.config`
- `configs/linux_rv64.config`
- `buildroot_rv64/output/build/linux-6.19.10/drivers/firmware/efi/libstub/efi-stub-entry.c`
- `buildroot_rv64/output/build/linux-6.19.10/drivers/firmware/efi/libstub/fdt.c`
- `buildroot_rv64/output/build/linux-6.19.10/arch/riscv/kernel/head.S`
- `dt/qemu_virt_rv64_mc.dts`
- `tools/riscv_uefi_symbols.py`
- `riscv-vp-plusplus/vp/src/core/common/gdb-mc/gdb_server.cpp`

## 9. 一句话结论

这次适配的本质，不是单独修复某一个启动错误，而是把 GUI-VP Kit 上原本分散、脆弱、难定位的 `OpenSBI -> UEFI -> Linux -> init` 启动链和 `OpenSBI + UEFI + Kernel` 调试链，改造成一条标准化、可观测、可源码级调试、可持续演进的完整工程路径。