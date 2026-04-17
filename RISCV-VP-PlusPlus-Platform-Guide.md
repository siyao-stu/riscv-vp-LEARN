# RISC-V VP++ QEMU-Virt 平台实现原理详解

> 本文档详细介绍 `riscv-vp-plusplus/vp/src/platform/qemu_virt` 路径下的平台代码实现原理。

## 1. 概述

`qemu_virt` 平台是 RISC-V VP++ (Virtual Prototype Plus Plus) 项目中一个模拟 QEMU virt 机器的虚拟平台实现。该平台旨在运行完整的操作系统（如 FreeBSD、CheriBSD），支持以下配置：

| 可执行文件 | 架构 | 核心数 | 特性 |
|-----------|------|--------|------|
| `qemu_virt32-sc-vp` | RV32 | 1 | 单核 |
| `qemu_virt32-mc-vp` | RV32 | 4 | 多核 |
| `qemu_virt64-sc-vp` | RV64 | 1 | 单核 |
| `qemu_virt64-mc-vp` | RV64 | 4 | 多核 |
| `qemu_virt64-cheriv9-sc-vp` | RV64 + CHERIv9 | 1 | CHERI 能力支持 |

---

## 2. 系统架构

### 2.1 内存映射 (Memory Map)

```
地址范围                    | 组件
---------------------------|---------------------------
0x00001000 - 0x00002FFF    | DTB ROM (设备树)
0x00100000 - 0x00100FFF    | SIFIVE_Test (系统控制)
0x00101000 - 0x00101FFF    | RTC (实时时钟, Dummy)
0x02000000 - 0x0200FFFF    | CLINT (核心本地中断控制器)
0x04000000 - 0x05FFFFFF    | Platform Bus (Dummy)
0x0C000000 - 0x0C5FFFFF    | PLIC (平台级中断控制器)
0x10000000 - 0x100000FF    | UART0 (NS16550A)
0x10001000 - 0x10008FFF    | VirtIO MMIO (Dummy)
0x10100000 - 0x10100017    | FW-CFG (Dummy)
0x20000000 - 0x21FFFFFF    | Flash0 (UEFI Code, 32MB)
0x22000000 - 0x23FFFFFF    | Flash1 (UEFI Vars, 32MB)
0x30000000 - 0x3FFFFFFF    | PCI (Dummy)
0x80000000 - ...           | 主内存 (默认 2GB)
```

### 2.2 核心组件关系图

```
                    ┌─────────────────────────────────────────────┐
                    │                SimpleBus                     │
                    │  (NUM_CORES + 1 initiators, 13 targets)     │
                    └─────────────────────────────────────────────┘
                              ▲                    │
                              │                    ▼
    ┌──────────────────────────────────────────────────────────────────┐
    │                         Core[0..N]                                │
    │  ┌─────┐   ┌─────┐   ┌─────────────────────┐   ┌──────────────┐ │
    │  │ ISS │◄──│ MMU │◄──│ CombinedMemoryIF    │◄──│ InstrMemProxy│ │
    │  └─────┘   └─────┘   └─────────────────────┘   └──────────────┘ │
    └──────────────────────────────────────────────────────────────────┘
                              │
    ┌─────────────────────────┴─────────────────────────────────────────┐
    │                           外设                                      │
    │  ┌────────┐ ┌──────┐ ┌──────────┐ ┌────────┐ ┌─────────────────┐ │
    │  │ Memory │ │ PLIC │ │  CLINT   │ │ UART   │ │ SIFIVE_Test    │ │
    │  └────────┘ └──────┘ └──────────┘ └────────┘ └─────────────────┘ │
    └───────────────────────────────────────────────────────────────────┘
```

---

## 3. 主要代码结构分析

### 3.1 `main.cpp` - 平台入口

#### 3.1.1 条件编译架构选择

```cpp
#if defined(TARGET_RV32)
#include "core/rv32/iss.h"
// ...
#elif defined(TARGET_RV64)
#include "core/rv64/iss.h"
// ...
#elif defined(TARGET_RV64_CHERIV9)
#include "core/rv64_cheriv9/iss.h"
// ...
#endif
```

通过预处理宏 `TARGET_RV32`、`TARGET_RV64`、`TARGET_RV64_CHERIV9` 选择不同的 ISS（指令集模拟器）实现。

#### 3.1.2 `LinuxOptions` 配置类

```cpp
struct LinuxOptions : public Options {
    addr_t mem_start_addr = 0x80000000;      // 内存起始地址
    addr_t mem_size = 2048MB;                 // 内存大小
    addr_t uart0_start_addr = 0x10000000;    // UART 地址
    addr_t plic_start_addr = 0xc000000;      // PLIC 地址
    addr_t clint_start_addr = 0x02000000;    // CLINT 地址
    // ... 更多外设地址配置
    
    std::string dtb_file;        // 设备树文件（必需）
    std::string kernel_file;     // 内核文件（可选）
    std::string uefi_code_image; // UEFI 代码映像
    // ...
};
```

#### 3.1.3 `Core` 类 - CPU 核心封装

```cpp
class Core {
public:
    ISS iss;                          // 指令集模拟器
    MMU mmu;                          // 内存管理单元
    CombinedMemoryInterface memif;    // 内存接口
    InstrMemoryProxy imemif;          // 指令内存代理（支持 DMI）

    Core(RV_ISA_Config *isa_config, unsigned int id, MemoryDMI dmi, 
         uint64_t mem_start_addr, uint64_t mem_end_addr)
        : iss(isa_config, id),
          mmu(iss),
          memif(("MemoryInterface" + std::to_string(id)).c_str(), iss, &mmu),
          imemif(dmi, iss) { }
          
    void init(bool use_data_dmi, bool use_instr_dmi, bool use_dbbcache, 
              bool use_lscache, clint_if *clint, uint64_t entry, uint64_t sp_base);
};
```

#### 3.1.4 `sc_main` - SystemC 主函数

主要执行流程：

```cpp
int sc_main(int argc, char **argv) {
    // 1. 解析命令行参数
    LinuxOptions opt;
    opt.parse(argc, argv);
    
    // 2. 创建系统总线
    SimpleBus<NUM_CORES + 1, 13> bus("SimpleBus", debug_bus, opt.break_on_transaction);
    
    // 3. 实例化外设
    SimpleMemory dtb_rom("DTB_ROM", opt.dtb_rom_size);
    SimpleMemory mem("SimpleMemory", opt.mem_size);  // 或 TaggedMemory for CHERI
    NS16550A_UART uart0_ns16550a("uart0", &channel_console, 10);
    SIFIVE_PLIC plic("PLIC", false, NUM_CORES, 96);
    LWRT_CLINT<NUM_CORES> clint("CLINT");
    SIFIVE_Test sifive_test("SIFIVE_Test");
    // ... 更多外设
    
    // 4. 创建 CPU 核心
    Core *cores[NUM_CORES];
    for (unsigned i = 0; i < NUM_CORES; i++) {
        cores[i] = new Core(&isa_config, i, dmi, opt.mem_start_addr, opt.mem_end_addr);
        cores[i]->memif.bus_lock = bus_lock;
        cores[i]->mmu.mem = &cores[i]->memif;
    }
    
    // 5. 配置端口映射
    bus.ports[i++] = new PortMapping(opt.mem_start_addr, opt.mem_end_addr, mem);
    bus.ports[i++] = new PortMapping(opt.plic_start_addr, opt.plic_end_addr, plic);
    // ... 更多映射
    
    // 6. 连接 TLM 套接字
    for (size_t i = 0; i < NUM_CORES; i++) {
        cores[i]->memif.isock.bind(bus.tsocks[i]);
    }
    bus.isocks[i++].bind(mem.tsock);
    // ... 更多绑定
    
    // 7. 连接中断信号
    for (size_t i = 0; i < NUM_CORES; i++) {
        plic.target_harts[i] = &cores[i]->iss;
        clint.target_harts[i] = &cores[i]->iss;
    }
    uart0_ns16550a.plic = &plic;
    
    // 8. 加载程序和设备树
    loader.load_executable_image(mem, mem.get_size(), opt.mem_start_addr);
    dtb_rom.load_binary_file(opt.dtb_file, 0);
    
    // 9. 设置启动参数（模拟 RISC-V boot loader）
    for (size_t i = 0; i < NUM_CORES; i++) {
        cores[i]->iss.regs[RegFile::a0] = cores[i]->iss.get_hart_id();  // hart id
        cores[i]->iss.regs[RegFile::a1] = opt.dtb_rom_start_addr;        // dtb 地址
    }
    
    // 10. 启动仿真
    sc_core::sc_start();
}
```

---

## 4. 关键外设实现分析

### 4.1 LWRT_CLINT - 轻量级实时 CLINT

```cpp
template <unsigned NumberOfCores>
struct LWRT_CLINT : public clint_if, public sc_core::sc_module {
    // 寄存器映射
    RegisterRange regs_mtime{0xBFF8, 8};       // mtime 寄存器
    RegisterRange regs_mtimecmp{0x4000, 8 * NumberOfCores};  // 每核一个 mtimecmp
    RegisterRange regs_msip{0x0, 4 * NumberOfCores};         // 软件中断寄存器
    
    std::array<clint_interrupt_target *, NumberOfCores> target_harts{};
```

**核心特点：**
- **实时时钟**：`mtime` 基于主机实际墙钟时间（`std::chrono::high_resolution_clock`），而非仿真时间
- **定时器中断**：当 `mtime >= mtimecmp` 且 `mtimecmp != 0` 时触发中断
- **软件中断**：通过 `msip` 寄存器触发/清除

```cpp
uint64_t update_and_get_mtime() override {
    auto now = get_time();  // 获取主机真实时间
    if (now > mtime)
        mtime = now;
    return mtime;
}

void run() {
    while (true) {
        sc_core::wait(10, sc_core::SC_US);  // 10us 轮询
        update_and_get_mtime();
        for (unsigned i = 0; i < NumberOfCores; ++i) {
            if (mtimecmp[i] > 0 && mtime >= mtimecmp[i]) {
                target_harts[i]->trigger_timer_interrupt();
            }
        }
    }
}
```

### 4.2 SIFIVE_PLIC - 平台级中断控制器

```cpp
struct SIFIVE_PLIC : public sc_core::sc_module, public interrupt_gateway {
    const unsigned NUMIRQ;  // 中断数量（96）
    
    // 寄存器
    ArrayView<uint32_t> interrupt_priorities;  // 中断优先级
    ArrayView<uint32_t> pending_interrupts;    // 待处理中断
    
    std::vector<external_interrupt_target *> target_harts{};  // 目标 hart
```

**功能：**
- 管理最多 96 个外部中断源
- 支持 M-Mode 和 S-Mode 中断
- 提供优先级仲裁和中断使能控制
- 通过 `gateway_trigger_interrupt(uint32_t irq)` 接口接收中断请求

### 4.3 NS16550A_UART - 串口控制器

```cpp
class NS16550A_UART : public sc_core::sc_module {
    static constexpr unsigned UART_RX_FIFO_DEPTH = 28;
    static constexpr unsigned UART_TX_FIFO_DEPTH = 24;
    
    // 寄存器地址
    static constexpr uint8_t RX_TX_DLL_REG_ADDR = 0x0;  // 收发/波特率低位
    static constexpr uint8_t IER_DLM_REG_ADDR = 0x1;    // 中断使能/波特率高位
    static constexpr uint8_t IIR_FCR_REG_ADDR = 0x2;    // 中断标识/FIFO控制
    static constexpr uint8_t LCR_REG_ADDR = 0x3;        // 线控制
    static constexpr uint8_t MCR_REG_ADDR = 0x4;        // 调制解调控制
    static constexpr uint8_t LSR_REG_ADDR = 0x5;        // 线状态
    static constexpr uint8_t MSR_REG_ADDR = 0x6;        // 调制解调状态
    static constexpr uint8_t SCR_REG_ADDR = 0x7;        // 暂存
    
    interrupt_gateway *plic = nullptr;  // 连接到 PLIC
    Channel_IF *channel;                 // 输入/输出通道
```

**特点：**
- 兼容 NS16550A 标准
- 支持 FIFO 模式
- 通过 `Channel_Console` 连接到主机终端
- 可触发 PLIC 中断（IRQ 10）

### 4.4 SimpleMemory - 简单内存模型

```cpp
struct SimpleMemory : public sc_core::sc_module, public load_if {
    uint8_t *data;       // 内存数据
    uint64_t size;       // 内存大小
    
    // TLM 传输接口
    void transport(tlm::tlm_generic_payload &trans, sc_core::sc_time &delay) {
        transport_dbg(trans);
        delay += access_delay;
    }
    
    // 直接内存接口 (DMI)
    bool get_direct_mem_ptr(tlm::tlm_generic_payload &trans, tlm::tlm_dmi &dmi) {
        dmi.set_dmi_ptr(data);
        dmi.allow_read_write();
        return true;
    }
```

**优化：**
- 支持 TLM DMI（Direct Memory Interface）用于快速内存访问
- 可配置访问延迟

### 4.5 SIFIVE_Test - 系统控制

```cpp
class SIFIVE_Test : public sc_core::sc_module {
    // 控制寄存器
    uint32_t reg_ctrl = 0;
```

**功能：**
- 写入 `0x5555` → 关机 (poweroff)
- 写入 `0x7777` → 重启 (reboot)

### 4.6 DUMMY_TLM_TARGET - 占位外设

```cpp
class DUMMY_TLM_TARGET : public sc_core::sc_module {
    void transport(tlm::tlm_generic_payload &trans, sc_core::sc_time &delay) {
        if (trans.get_command() == tlm::TLM_WRITE_COMMAND) {
            // 忽略写操作
        } else if (trans.get_command() == tlm::TLM_READ_COMMAND) {
            // 读返回 0
            memcpy(trans.get_data_ptr(), &val, len);
        }
    }
```

用于模拟尚未实现的外设（如 PCI、VirtIO、RTC、FW-CFG）。

### 4.7 MemoryMappedFile - 文件映射存储

```cpp
struct MemoryMappedFile : public sc_core::sc_module {
    string mFilepath;      // 映射的主机文件路径
    uint32_t mSize;        // 存储区域大小
    fstream file;          // 文件流对象
    bool emulate_cfi_nor;  // 是否模拟 CFI NOR Flash
```

**特点：**
- 将主机文件系统中的文件直接映射为虚拟平台的存储空间
- 支持**持久化存储**：对 Flash 的写入会同步到主机文件
- 支持 **CFI NOR Flash** 模拟（编程只能将位从 1 变为 0，擦除将整个块恢复为全 1）

---

## 5. 系统总线实现

### 5.1 SimpleBus

```cpp
template <unsigned int NR_OF_INITIATORS, unsigned int NR_OF_TARGETS>
struct SimpleBus : sc_core::sc_module {
    std::array<tlm_utils::simple_target_socket<SimpleBus>, NR_OF_INITIATORS> tsocks;
    std::array<tlm_utils::simple_initiator_socket<SimpleBus>, NR_OF_TARGETS> isocks;
    std::array<PortMapping *, NR_OF_TARGETS> ports;
    
    int decode(uint64_t addr) {
        for (unsigned i = 0; i < NR_OF_TARGETS; ++i) {
            if (ports[i]->contains(addr))
                return i;
        }
        return -1;  // 地址错误
    }
    
    void transport(tlm::tlm_generic_payload &trans, sc_core::sc_time &delay) {
        auto addr = trans.get_address();
        auto id = decode(addr);
        
        trans.set_address(ports[id]->global_to_local(addr));  // 转换地址
        isocks[id]->b_transport(trans, delay);  // 转发到目标
    }
```

### 5.2 BusLock - LR/SC 原子操作支持

```cpp
class BusLock : public bus_lock_if {
    bool locked = false;
    unsigned owner = 0;
    sc_core::sc_event lock_event;
    
    virtual void lock(unsigned hart_id) override {
        if (locked && (hart_id != owner)) {
            wait_until_unlocked();
        }
        locked = true;
        owner = hart_id;
    }
    
    virtual void unlock(unsigned hart_id) override {
        if (locked && (owner == hart_id)) {
            locked = false;
            lock_event.notify(sc_core::SC_ZERO_TIME);
        }
    }
```

确保 RISC-V 的 Load-Reserved/Store-Conditional (LR/SC) 原子指令语义正确。

---

## 6. 设备树生成器

`dt/qemu_virt-dt-gen.py` 是一个 Python 脚本，用于生成与平台配置匹配的设备树源文件 (DTS)。

### 6.1 使用方法

```bash
qemu_virt-dt-gen.py -t qemu_virt64-sc-vp -m 0x80000000 -s 2147483648 -o output.dts
```

### 6.2 生成的设备树结构

```dts
/dts-v1/;

/ {
    compatible = "riscv-virtio";
    model = "riscv-virtio,qemu";
    
    memory@80000000 {
        device_type = "memory";
        reg = <0x0 0x80000000 0x0 0x80000000>;  // 2GB @ 0x80000000
    };
    
    cpus {
        timebase-frequency = <1000000>;  // 1MHz
        cpu@0: cpu@0 {
            device_type = "cpu";
            compatible = "riscv";
            riscv,isa = "rv64imafdc_zicntr_zicsr_zifencei";
            mmu-type = "riscv,sv57";
            // ...
        };
    };
    
    soc {
        serial@10000000 {
            compatible = "ns16550a";
            interrupts = <0x0a>;
            interrupt-parent = <&plic>;
        };
        
        plic: plic@c000000 {
            compatible = "sifive,plic-1.0.0";
            riscv,ndev = <0x5f>;  // 95 个设备
        };
        
        clint@2000000 {
            compatible = "sifive,clint0";
        };
        
        test@100000 {
            compatible = "sifive,test1", "syscon";
        };
    };
};
```

---

## 7. 启动流程

### 7.1 OpenSBI + Kernel 启动

1. **固件加载**：将 OpenSBI (`fw_dynamic.bin`) 加载到 `0x80000000`
2. **DTB 配置**：设备树加载到 `0x1000`
3. **寄存器设置**：
   - `a0` = hart ID
   - `a1` = DTB 地址
   - `a2` = fw_dynamic_info 结构地址（用于 fw_dynamic）
4. **执行入口**：PC 设为 `0x80000000`（或命令行指定的 entry point）

### 7.2 FW_Dynamic_Info 结构

```cpp
struct FW_Dynamic_Info {
    unsigned long magic;       // 0x4946534F ("OSBI")
    unsigned long version;     // 2
    unsigned long next_addr;   // 下一阶段地址
    unsigned long next_mode;   // S-Mode (1) 或 M-Mode (3)
    unsigned long options;     // 选项
    unsigned long boot_hart;   // 启动 hart
};
```

---

## 8. 调试支持

### 8.1 GDB 服务器

```cpp
if (opt.use_debug_runner) {
    auto server = new GDBServer("GDBServer", dharts, &dbg_if, opt.debug_port, ...);
    for (size_t i = 0; i < dharts.size(); i++)
        new GDBServerRunner(("GDBRunner" + std::to_string(i)).c_str(), server, dharts[i]);
}
```

支持通过 GDB 远程调试仿真。

### 8.2 Channel_Console 交互式调试

```cpp
Channel_Console channel_console;
// ...
channel_console.debug_targets_add(&cores[i]->iss);
```

在运行时可通过控制台执行调试命令。

---

## 9. 性能优化技术

| 优化技术 | 说明 |
|---------|------|
| **DMI (Direct Memory Interface)** | 绕过 TLM 事务直接访问内存 |
| **指令缓存 (dbbcache)** | 缓存已解码的基本块 |
| **数据缓存 (lscache)** | 缓存 Load/Store 结果 |
| **LWRT_CLINT** | 使用主机实时时钟避免仿真时间同步开销 |

---

## 10. 总结

`qemu_virt` 平台是一个功能完整的 RISC-V 虚拟原型，其设计特点：

1. **模块化设计**：基于 SystemC/TLM-2.0，各组件可独立开发和测试
2. **兼容 QEMU virt 机器**：内存映射和外设与 QEMU 兼容，便于使用现有软件
3. **多核支持**：通过模板参数 `NUM_CORES` 支持多核配置
4. **可扩展性**：未实现的外设通过 `DUMMY_TLM_TARGET` 占位
5. **CHERI 支持**：可选的 CHERIv9 能力扩展用于内存安全研究
6. **实时性**：`LWRT_CLINT` 提供基于主机时钟的实时定时器

---

## 附录：文件结构

```
riscv-vp-plusplus/vp/src/platform/qemu_virt/
├── main.cpp                    # 平台主入口
├── CMakeLists.txt              # CMake 构建配置
├── README.md                   # 平台说明文档
└── dt/
    └── qemu_virt-dt-gen.py     # 设备树生成脚本
```

---

*文档生成日期：2026年4月17日*
