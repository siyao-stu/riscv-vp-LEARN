# RISC-V VP++ UEFI 适配修改总结

> 本文档总结了为 RISC-V VP++ 项目添加 UEFI 固件启动支持所做的代码修改。

## 1. 修改概览

该项目的本地修改共涉及 **4 个文件**，总计 **+343 行，-28 行**：

| 文件 | 新增行数 | 修改内容 |
|------|---------|----------|
| `memory_mapped_file.h` | +237 | CFI NOR Flash 模拟功能 |
| `qemu_virt/main.cpp` | +64 | UEFI 启动支持、fw_dynamic 处理 |
| `linux/linux_main.cpp` | +66 | Linux 平台同步 UEFI Flash 支持 |
| `qemu_virt-dt-gen.py` | +4 | 设备树启用 Flash 节点 |

---

## 2. 详细修改分析

### 2.1 `memory_mapped_file.h` - CFI NOR Flash 模拟功能

#### 修改目的

为 `MemoryMappedFile` 组件添加 **CFI (Common Flash Interface) NOR Flash** 的模拟功能，用于支持 UEFI 固件启动。UEFI 需要通过标准的 CFI Flash 接口来读写固件配置。

#### 主要新增内容

##### CFI 状态机

```cpp
enum class CfiMode { 
    ReadArray,      // 正常读取数组数据
    ReadStatus,     // 读取状态寄存器
    ReadDeviceId,   // 读取设备 ID
    ReadCfiQuery    // 读取 CFI 查询信息
};

enum class CfiProgramState { 
    None,                    // 空闲
    WordProgramPendingData,  // 等待字编程数据
    BlockErasePendingConfirm,// 等待块擦除确认
    BufferedPendingCount,    // 等待缓冲区计数
    BufferedPendingData,     // 等待缓冲区数据
    BufferedPendingConfirm   // 等待缓冲区编程确认
};
```

##### Flash 编程特性

```cpp
void program_word(unsigned addr, uint32_t data) {
    uint32_t old = 0xFFFFFFFFu;
    read_word(addr, old);
    uint32_t programmed = old & data;  // Flash 只能将 1 变为 0
    // 写入结果
}

void erase_block_containing(unsigned addr) {
    // 将整个块填充为 0xFF（擦除状态）
    unsigned base = (addr / cfi_block_size) * cfi_block_size;
    std::vector<uint8_t> erased(len, 0xFF);
    write_data(base, erased.data(), len);
}
```

##### CFI 命令支持

| 命令码 | 功能 | 说明 |
|--------|------|------|
| `0x00FF` | READ ARRAY | 切换到正常读模式 |
| `0x0070` | READ STATUS | 读取状态寄存器 |
| `0x0050` | CLEAR STATUS | 清除状态寄存器 |
| `0x0090` | READ DEVICE ID | 读取设备标识 |
| `0x0098` | READ CFI QUERY | 读取 CFI 查询表 |
| `0x0040/0x0010` | WORD PROGRAM | 字编程（单字写入） |
| `0x0020` | BLOCK ERASE | 块擦除 |
| `0x00E8` | BUFFERED PROGRAM | 缓冲区编程（批量写入） |
| `0x00D0` | CONFIRM | 确认擦除/编程操作 |

##### Bug 修复

```cpp
// 修复：写入时应使用 seekp 而非 seekg
- file.seekg(addr, file.beg);
+ file.seekp(addr, file.beg);
+ file.flush();  // 新增：立即刷新到磁盘

// 修复：读取失败后缺少 return
+ return;
```

#### 有效性评估

| 功能 | 有效性 |
|------|--------|
| CFI 状态机 | ✅ 有效 |
| Flash 编程特性 | ✅ 有效 |
| 块擦除功能 | ✅ 有效 |
| CFI 命令处理 | ✅ 有效 |
| 缓冲区编程 | ✅ 有效 |
| Bug 修复 | ✅ 有效 |

---

### 2.2 `qemu_virt/main.cpp` - UEFI 启动支持

#### 修改目的

为 qemu_virt 平台添加 UEFI 固件启动能力，支持两种 OpenSBI 启动模式：
- `fw_jump`：跳转到固定地址
- `fw_dynamic`：通过 `a2` 寄存器传递动态信息

#### 主要新增内容

##### FW_Dynamic_Info 结构体

```cpp
struct FW_Dynamic_Info {
    unsigned long magic;       // 0x4946534F ("OSBI")
    unsigned long version;     // 版本号
    unsigned long next_addr;   // 下一阶段地址
    unsigned long next_mode;   // S-Mode (1) 或 M-Mode (3)
    unsigned long options;     // 选项
    unsigned long boot_hart;   // 启动 hart
};

#define FW_DYNAMIC_INFO_MAGIC_VALUE      0x4946534F
#define FW_DYNAMIC_INFO_VERSION_2        0x2
#define FW_DYNAMIC_INFO_NEXT_MODE_S      0x1
#define FW_DYNAMIC_INFO_NEXT_MODE_M      0x3
```

##### UEFI Flash 映射

```cpp
// 原来使用 DUMMY_TLM_TARGET
- DUMMY_TLM_TARGET flash0("flash0", opt.flash0_size);
- DUMMY_TLM_TARGET flash1("flash1", opt.flash1_size);

// 修改为 MemoryMappedFile
+ MemoryMappedFile flash0("flash0", opt.uefi_code_image, opt.flash0_size, true);
+ MemoryMappedFile flash1("flash1", opt.uefi_vars_image, opt.flash1_size, true);
```

##### 命令行参数

```cpp
("uefi-code-image", po::value<std::string>(&uefi_code_image), "UEFI code flash image file")
("uefi-vars-image", po::value<std::string>(&uefi_vars_image), "UEFI vars flash image file")
```

##### fw_dynamic 处理函数

```cpp
void handle_fw_dynamic_info(const LinuxOptions &opt, load_if &mem, Core *cores[NUM_CORES]) {
    if (!is_fw_dynamic_image(opt.input_program)) {
        return;
    }

    // 设置下一阶段地址和模式
    unsigned long next_stage_addr = opt.flash0_start_addr;  // 默认: Flash 地址
    unsigned long next_stage_mode = FW_DYNAMIC_INFO_NEXT_MODE_S;

    if (!opt.uefi_code_image.empty()) {
        // UEFI 模式：固件需要在 M-mode 从 RAM 运行
        next_stage_mode = FW_DYNAMIC_INFO_NEXT_MODE_M;

        // 如果没有指定 kernel_file，需要自动加载 UEFI 到 RAM
        if (opt.kernel_file.empty()) {
            mem.load_binary_file(opt.uefi_code_image, KERNEL_LOAD_ADDR - opt.mem_start_addr);
        }
        // 设置 next_addr 为 UEFI 在 RAM 中的加载地址
        next_stage_addr = KERNEL_LOAD_ADDR;
    }

    // 创建并写入 FW_Dynamic_Info 结构
    FW_Dynamic_Info dyn_info = {
        .magic = FW_DYNAMIC_INFO_MAGIC_VALUE,
        .version = FW_DYNAMIC_INFO_VERSION_2,
        .next_addr = next_stage_addr,
        .next_mode = next_stage_mode,
        .boot_hart = (unsigned long)NUM_CORES - 1UL,
    };
    
    // 设置所有核心的 a2 寄存器
    for (size_t i = 0; i < NUM_CORES; i++) {
        cores[i]->iss.regs[RegFile::a2] = dyn_info_addr;
    }
}
```

#### 有效性评估

| 功能 | 有效性 |
|------|--------|
| FW_Dynamic_Info 结构体 | ✅ 有效 |
| UEFI Flash 映射 | ✅ 有效 |
| 命令行参数 | ✅ 有效 |
| handle_fw_dynamic_info() | ✅ 有效 |
| Flash 大小验证 | ✅ 有效 |

---

### 2.3 `linux/linux_main.cpp` - Linux 平台同步更新

#### 修改目的

将 UEFI Flash 支持同步添加到 linux 平台，保持代码一致性。

#### 主要变更

| 变更 | 说明 | 有效性 |
|------|------|--------|
| 总线端口扩展 | 从 20 端口扩展到 23 端口 | ✅ 有效 |
| UEFI Flash 组件 | 添加 `uefiCodeFlash` 和 `uefiVarsFlash` | ✅ 有效 |
| 命令行参数 | 与 qemu_virt 保持一致 | ✅ 有效 |
| PLIC 地址范围修正 | `0x10000000` → `0x0FFFFFFF` | ✅ 有效 |
| platform_bus | 新增 dummy 占位设备 | ✅ 有效 |

---

### 2.4 `dt/qemu_virt-dt-gen.py` - 设备树启用 Flash

#### 修改内容

```python
# 移除注释，启用 flash 节点
- /*
- 	// NOT SUPPORTED IN RISC-V VP++
 	flash@20000000 {
 		bank-width = <0x04>;
 		reg = <0x00 0x20000000 0x00 0x2000000 0x00 0x22000000 0x00 0x2000000>;
 		compatible = "cfi-flash";
+		status = "okay";
 	};
- */
```

#### 有效性评估

✅ 有效 - 与代码修改配套，启用设备树中的 Flash 节点。

---

## 3. Makefile 启动目标

### 3.1 `run_qemu_virt64_mc_uefi`（已调通）

```makefile
run_qemu_virt64_mc_uefi: .stamp/vp_build .stamp/buildroot_rv64_build dt/qemu_virt_rv64_mc.dtb
	$(VP_NAME)/vp/build/bin/qemu_virt64-mc-vp					\
		$(QEMU_VP_ARGS)								\
		--dtb-file=dt/qemu_virt_rv64_mc.dtb					\
 		--kernel-file $(UEFI_CODE_IMAGE)					\
		--uefi-code-image $(UEFI_CODE_IMAGE)					\
		--uefi-vars-image $(UEFI_VARS_IMAGE)					\
		--memory-size $(MEM_SIZE_RV64)						\
		--debug-cont-sim-on-wait                                		\
		buildroot_rv64/output/images/fw_jump.elf
```

### 3.2 `run_qemu_virt64_mc_uefi_dynamic`

```makefile
run_qemu_virt64_mc_uefi_dynamic: .stamp/vp_build .stamp/buildroot_rv64_build dt/qemu_virt_rv64_mc.dtb
	$(VP_NAME)/vp/build/bin/qemu_virt64-mc-vp					\
		$(QEMU_VP_ARGS)								\
		--dtb-file=dt/qemu_virt_rv64_mc.dtb					\
		--uefi-code-image $(UEFI_CODE_IMAGE)					\
		--uefi-vars-image $(UEFI_VARS_IMAGE)					\
		--memory-size $(MEM_SIZE_RV64)						\
		--debug-cont-sim-on-wait						\
		buildroot_rv64/output/images/fw_dynamic.elf
```

### 3.3 两种启动方式对比

| 参数 | `fw_jump` 模式 | `fw_dynamic` 模式 |
|------|---------------|-------------------|
| OpenSBI 固件 | `fw_jump.elf` | `fw_dynamic.elf` |
| `--kernel-file` | ✅ 由 VP 加载器处理 | ❌ 由代码自动加载 |
| `--debug-cont-sim-on-wait` | ✅ | ✅ |
| UEFI 加载位置 | RAM `0x80200000` | RAM `0x80200000` |
| 跳转地址来源 | 硬编码在 OpenSBI | `fw_dynamic_info.next_addr` |

---

## 4. 启动流程

### 4.1 fw_jump 模式

```
┌─────────────────────────────────────────────────────────────────┐
│                     fw_jump 启动流程                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. VP 加载 fw_jump.elf 到 0x80000000                           │
│  2. VP 通过 --kernel-file 加载 UEFI 到 0x80200000               │
│  3. 设置 a0=hart_id, a1=dtb_addr                                │
│  4. PC = 0x80000000 (OpenSBI 入口)                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. OpenSBI 初始化                                              │
│  6. 跳转到硬编码地址 0x80200000 (UEFI 入口)                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. UEFI 执行                                                   │
│  8. 启动到 UEFI Shell                                           │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 fw_dynamic 模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    fw_dynamic 启动流程                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. VP 加载 fw_dynamic.elf 到 0x80000000                        │
│  2. handle_fw_dynamic_info() 自动加载 UEFI 到 0x80200000        │
│  3. 创建 FW_Dynamic_Info 结构并写入内存                          │
│  4. 设置 a0=hart_id, a1=dtb_addr, a2=dyn_info_addr              │
│  5. PC = 0x80000000 (OpenSBI 入口)                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. OpenSBI 读取 a2 指向的 FW_Dynamic_Info                       │
│  7. 获取 next_addr=0x80200000, next_mode=M-mode                 │
│  8. 跳转到 0x80200000 (UEFI 入口)                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  9. UEFI 执行                                                   │
│  10. 启动到 UEFI Shell                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. 代码有效性总结

### 5.1 有效代码统计

| 文件 | 新增行数 | 有效代码 | 问题代码 |
|------|---------|---------|---------|
| `memory_mapped_file.h` | +237 | ~237 行 | 无 |
| `qemu_virt/main.cpp` | +64 | 64 行 | 无 |
| `linux/linux_main.cpp` | +66 | 66 行 | 无 |
| `qemu_virt-dt-gen.py` | +4 | 4 行 | 无 |

### 5.2 结论

**所有修改代码均为有效代码**，主要实现了以下功能：

1. ✅ **CFI NOR Flash 模拟**：完整实现了 UEFI 所需的 Flash 接口
2. ✅ **UEFI 启动支持**：支持通过 Flash 映像启动 UEFI 固件
3. ✅ **fw_dynamic 模式**：实现了 OpenSBI fw_dynamic 协议支持
4. ✅ **设备树更新**：启用了 Flash 设备节点
5. ✅ **Bug 修复**：修复了文件读写的多个问题

---

## 6. 运行验证

### 6.1 fw_jump 模式（已验证通过）

```bash
cd /home/wangsiyao/riscv-dev/code/GUI-VP_Kit
make run_qemu_virt64_mc_uefi
```

预期输出：
```
        SystemC 3.0.1-Accellera --- Apr 13 2026 18:51:26
CONSOLE: press ^A-h for help
Info: load kernel file "edk2/.../RISCV_VIRT_CODE.fd" as RAW file (to 0x80200000)

OpenSBI v1.6
   ____                    _____ ____ ___
  / __ \                  / ____|  _ \_ _|
 | |  | |_ __   ___ _ __ | (___ | |_) | |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |
  \____/| .__/ \___|_| |_|_____/|____/___|
        | |
        |_|

...

UEFI Interactive Shell v2.2
Shell>
```

### 6.2 fw_dynamic 模式

```bash
cd /home/wangsiyao/riscv-dev/code/GUI-VP_Kit
make run_qemu_virt64_mc_uefi_dynamic
```

预期输出：
```
        SystemC 3.0.1-Accellera --- Apr 13 2026 18:51:26
CONSOLE: press ^A-h for help
Info: fw_dynamic mode - loading UEFI image to RAM at 0x80200000
Info: fw_dynamic_info - next_addr=0x80200000, next_mode=3, boot_hart=3

OpenSBI v1.6
...

UEFI Interactive Shell v2.2
Shell>
```

---

## 7. 注意事项

1. **UEFI 固件是位置相关的**：EDK2 UEFI 编译时按照 RAM 地址 `0x80200000` 设计，无法直接从 Flash XIP 执行
2. **多核模式需要 `--debug-cont-sim-on-wait`**：否则 WFI 指令会阻塞整个 SystemC 仿真
3. **Flash 持久化**：对 UEFI 变量的修改会保存到主机文件系统

---

*文档生成日期：2026年4月17日*
