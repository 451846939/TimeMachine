# hvisor硬件抽象





关于不同平台架构相关抽象的函数




| 模块分类         | 功能描述             | 接口 / Trait 名                             | RISC-V 实现方式                                            |
| ---------------- | -------------------- | ------------------------------------------- | ---------------------------------------------------------- |
| **分页系统**     | 页表项抽象           | `GenericPTE`                                | `PageTableEntry` 实现 `addr/flags/is_unused/...`           |
|                  | 页表操作接口         | `GenericPageTable`, `GenericPageTableImmut` | `Level3PageTable<HostPhysAddr, PageTableEntry, S1PTInstr>` |
|                  | 页表激活与刷新       | `PagingInstr`                               | `S1PTInstr::activate()` 操作 SATP                          |
|                  | 页大小与页操作       | `PageSize`, `Page<VA>`                      | 枚举支持 4K/2M/1G，提供对齐/偏移等方法                     |
| **异常与中断**   | 安装异常向量表       | `install_trap_vector()`                     | 设置 `stvec = _hyp_trap_vector`，跳转到汇编入口            |
|                  | Trap 返回处理        | `_hyp_trap_return()`                        | 汇编 `sret` 实现                                           |
|                  | 注入虚拟中断         | `inject_irq()`                              | `plic::set_pending()`                                      |
|                  | 中断控制器早期初始化 | `primary_init_early()`                      | `plic::init()` 初始化优先级和使能表                        |
|                  | 中断控制器晚期初始化 | `primary_init_late()`                       | `plic::set_threshold()` 设置中断阈值                       |
|                  | 每核中断初始化       | `percpu_init()`                             | `plic::hart_init()`                                        |
| **IPI**          | 向目标核发送 IPI     | `arch_send_event(cpu_id, sgi_num)`          | 使用 `sbi_send_ipi()` 调用 SBI 扩展                        |
| **CPU 启动流程** | 启动其他 CPU 核      | `cpu_start(cpu_id, entry_fn, dtb)`          | 设置 `sepc`、栈，使用 IPI 唤醒核                           |
|                  | 架构启动入口         | `arch_entry()`                              | 汇编入口跳转 `rust_main`，初始化 trap、页表等              |
| **UART 控制台**  | 输出单个字符         | `console_putchar(ch)`                       | 调用 `sbi_call_legacy_1(1, ch)`（LEGACY_CONSOLE_PUTCHAR）  |
|                  | 读取单个字符         | `console_getchar()`                         | 调用`sbi_call_legacy_0(2)`（LEGACY_CONSOLE_GETCHAR）       |
| **内存初始化**   | 初始化 HV 页表       | `init_hv_page_table(fdt)`                   | 从设备树构造初始 S1 页表结构                               |
|                  | 创建 Stage-2 页表    | `new_s2_memory_set()`                       | 创建 `Stage2PageTable`，映射 Guest IPA                     |
|                  | 页表区域初始化       | `zone::pt_init()`                           | 初始化页表所需区域                                         |
|                  | IRQ bitmap 初始化    | `zone::irq_bitmap_init()`                   | 分配并清零 IRQ bitmap                                      |

## trap

在hvisor中`src/arch/riscv64/trap.S` 中可以大概总结为以下流程

```text
                     +-----------------------+
                     |    trap occurs (VS-mode)
                     +-----------------------+
                               |
                         -->  _hyp_trap_vector:
                               |
                            VM_EXIT       ; 保存上下文
                               |
                        csrr t0, scause  ; 获取 trap 原因
                               |
                     +---------+---------+
                     |                   |
           t0 >= 0 (Exception)   t0 < 0 (Interrupt)
                     |                   |
        call sync_exception_handler   call interrupts_arch_handle
                     |
                  (返回)
                               |
                         --> VM_ENTRY
                               |
                           sret -> 回到 Guest
```



我们先看一下risc-v的`_hyp_trap_vector`

```assembly
.balign 0x4
.global _hyp_trap_vector	
_hyp_trap_vector:
    VM_EXIT
    csrr    t0, scause
    bltz    t0, 1f
    call    {sync_exception_handler}
    j       2f
1:
    call    {interrupts_arch_handle}
2:
.global vcpu_arch_entry
vcpu_arch_entry:
    VM_ENTRY
```

- scause 是 RISC-V 中断/异常来源寄存器：

  - MSB（bit 63）= 1 → 中断（Interrupt）
  - MSB（bit 63）= 0 → 异常（Exception）

1. 统一保存 Guest 的寄存器状态（VM_EXIT）。

2. 读取 scause 判断是 中断 还是 异常。

   •	是异常：调用 sync_exception_handler。
   •	是中断：调用 interrupts_arch_handle，进一步 dispatch 到 SSI/STI/SEI。

3. 最终由 vcpu_arch_entry 调用 VM_ENTRY 恢复寄存器并 sret 返回 Guest。  

### interrupts_arch_handle 

再看下risc-v的`interrupts_arch_handle` 中断是如何处理的

```rust
pub fn interrupts_arch_handle(current_cpu: &mut ArchCpu) {
    let trap_code: usize;
    trap_code = read_csr!(CSR_SCAUSE);
    trace!("CSR_SCAUSE: {:#x}", trap_code);
    match trap_code & 0xfff {
        InterruptType::STI => {
            unsafe {
                hvip::set_vstip();
                sie::clear_stimer();
            }
        }
        InterruptType::SSI => {
            handle_ssi(current_cpu);
        }
        InterruptType::SEI => {
            handle_eirq(current_cpu)
        }
        _ => {
            error!(
                "unhandled trap {:#x},sepc: {:#x}",
                trap_code, current_cpu.sepc
            );
            unreachable!();
        }
    }
}
```

这里定义了3类中断

- SSI（Supervisor Software Interrupt）软件中断常用于多核间 IPI（Inter-Processor Interrupt）
- STI（Supervisor Timer Interrupt） 定时器中断
- SEI（Supervisor External Interrupt）外部设备的中断，比如串口、网卡等外设产生的中断。

1. handle_ssi
   - 从 S-mode 的 SIP 中清除 bit1（host software interrupt）
   - 将 HVIP 的 bit2 置位 → **注入虚拟 software interrupt**
   - 这是 **IPI 跨核通信的中断机制**
2. handle_eirq
   - 通过 PLIC 的 claim 接口获取外部中断号
   - 调用 hvip::set_vseip()，设置 HVIP[10]：**注入虚拟 external interrupt 给 Guest**
3. interruptType::STI
   - hvip::set_vstip()：设置 HVIP[6]，让 guest 收到 timer 中断
   - sie::clear_stimer()：取消 host 当前核 timer 中断（避免重复触发）

### sync_exception_handler

再看下exception是怎么处理的

```rust
pub fn sync_exception_handler(current_cpu: &mut ArchCpu) {
    let trap_code = read_csr!(CSR_SCAUSE);
    let trap_pc = read_csr!(CSR_SEPC);     // Guest 的 PC
    let trap_value = read_csr!(CSR_HTVAL); // 地址类异常的附加信息
    let trap_ins = read_csr!(CSR_HTINST);  // 异常指令内容
    // 若不是从 VS（虚拟 supervisor）模式 trap 来的，报错
    if (read_csr!(CSR_HSTATUS) & (1 << 7)) == 0 {
        error!("Exception from HS mode (not from guest VS)");
        return;
    }
    match trap_code {
        ExceptionType::ECALL_VU => {
            error!("Unhandled ECALL from U-mode guest");
        }
        ExceptionType::ECALL_VS => {
            trace!("ECALL from S-mode guest");
            sbi_vs_handler(current_cpu);
            current_cpu.sepc += 4; // 跳过 ECALL 指令
        }
        ExceptionType::LOAD_GUEST_PAGE_FAULT |
        ExceptionType::STORE_GUEST_PAGE_FAULT => {
            trace!("Guest Page Fault");
            guest_page_fault_handler(current_cpu);
        }
        _ => {
            let raw_inst = read_inst(trap_pc);
            let inst = riscv_decode::decode(raw_inst);
            current_cpu.idle(); // 进入 idle，避免继续执行
        }
    }
}
```

1. ECALL：
   - 来自虚拟 U/S 模式的 ecall 指令
   - ECALL_VS → 转发给 SBI handler
   - ECALL_VU → 打印错误，未处理
2. 页错误：
   - 读/写触发 Guest 页表缺页异常 → 调用 guest_page_fault_handler
3. 其他异常：
   - 打印调试信息：scause、htval、htinst、sepc、指令内容
   - 默认 idle() 掉该 vCPU，避免系统崩溃或错误执行







分界线

---

未完，下面是看代码整理的草稿和结合GPT整理的资料，可以用作参考



草稿：

```rust
arch::trap::install_trap_vector()
ArchCpu
cpu_start(cpu_id, arch_entry as _, host_dtb)

ipi 
arch_send_event(cpu_id: u64, _sgi_num: u64)
mm
init_hv_page_table(fdt: &fdt::Fdt)
new_s2_memory_set() -> MemorySet<Stage2PageTable>
paging
统一抽象了下面结构体与实现以及trait GenericPTE，但是没有提出去成为公用trait
pub trait GenericPTE: Debug + Clone {
    /// Returns the physical address mapped by this entry.
    fn addr(&self) -> PhysAddr;
    /// Returns the flags of this entry.
    fn flags(&self) -> MemFlags;
    /// Returns whether this entry is zero.
    fn is_unused(&self) -> bool;
    /// Returns whether this entry flag indicates present.
    fn is_present(&self) -> bool;
    /// Returns whether this entry maps to a huge frame.
    fn is_huge(&self) -> bool;
    /// Set physical address for terminal entries.
    fn set_addr(&mut self, paddr: PhysAddr);
    /// Set flags for terminal entries.
    fn set_flags(&mut self, flags: MemFlags, is_huge: bool);
    /// Set physical address and flags for intermediate table entries.
    fn set_table(&mut self, paddr: PhysAddr);
    /// Set this entry to zero.
    fn clear(&mut self);
}
#[repr(usize)]
#[derive(Debug, Copy, Clone, Eq, PartialEq)]
pub enum PageSize {
    Size4K = 0x1000,
    Size2M = 0x20_0000,
    Size1G = 0x4000_0000,
}

#[derive(Debug, Copy, Clone)]
pub struct Page<VA> {
    vaddr: VA,
    size: PageSize,
}
impl PageSize {
    pub const fn is_aligned(self, addr: usize) -> bool {
        self.page_offset(addr) == 0
    }

    pub const fn align_down(self, addr: usize) -> usize {
        addr & !(self as usize - 1)
    }

    pub const fn page_offset(self, addr: usize) -> usize {
        addr & (self as usize - 1)
    }

    pub const fn is_huge(self) -> bool {
        matches!(self, Self::Size1G | Self::Size2M)
    }
}
pub trait PagingInstr {
    unsafe fn activate(root_paddr: PhysAddr);
    fn flush(vaddr: Option<usize>);
}

/// A basic read-only page table for address query only.
pub trait GenericPageTableImmut: Sized {
    type VA: From<usize> + Into<usize> + Copy;

    unsafe fn from_root(root_paddr: PhysAddr) -> Self;
    fn root_paddr(&self) -> PhysAddr;
    fn query(&self, vaddr: Self::VA) -> PagingResult<(PhysAddr, MemFlags, PageSize)>;
}

/// A extended mutable page table can change mappings.
pub trait GenericPageTable: GenericPageTableImmut {
    fn new() -> Self;

    fn map(&mut self, region: &MemoryRegion<Self::VA>) -> HvResult;
    fn unmap(&mut self, region: &MemoryRegion<Self::VA>) -> HvResult;
    fn update(
        &mut self,
        vaddr: Self::VA,
        paddr: PhysAddr,
        flags: MemFlags,
    ) -> PagingResult<PageSize>;

    fn clone(&self) -> Self;

    unsafe fn activate(&self);
    fn flush(&self, vaddr: Option<Self::VA>);
}
#[derive(Debug)]
pub enum PagingError {
    NoMemory,
    NotMapped,
    AlreadyMapped,
    MappedToHugePage,
}

pub type PagingResult<T = ()> = Result<T, PagingError>;

不同的架构PageTable有不同的实现主要用于s1pt和s2pt比如在loongarch中封装的为
/// A extended level-4 page table implements `GenericPageTable`. It use locks to avoid data
/// racing between it and its clonees.
pub struct Level4PageTable<VA, PTE: GenericPTE, I: PagingInstr> {
    inner: Level4PageTableUnlocked<VA, PTE, I>,
    /// Make sure all accesses to the page table and its clonees is exclusive.
    clonee_lock: Arc<Mutex<()>>,
}
s1pt
pub type Stage1PageTable = Level4PageTable<GuestPhysAddr, PageTableEntry, S1PTInstr>;
s2pt
pub type Stage2PageTable = Level4PageTable<GuestPhysAddr, PageTableEntry, S2PTInstr>;



在risc-v封装为

pub struct Level3PageTable<VA, PTE: GenericPTE, I: PagingInstr> {
    inner: Level3PageTableUnlocked<VA, PTE, I>,
    /// Make sure all accesses to the page table and its clonees is exclusive.
    clonee_lock: Arc<Mutex<()>>,
}
s1pt
pub type Stage1PageTable = Level3PageTable<HostPhysAddr, PageTableEntry, S1PTInstr>;
s2pt
pub type Stage2PageTable = Level3PageTable<HostPhysAddr, PageTableEntry, S2PTInstr>;


s1pt
s2pt

zone
pt_init
mmio_init
irq_bitmap_init

trap
install_trap_vector
_hyp_trap_vector
_hyp_trap_return

entry
arch_entry


irq
主要针对不同的架构也做了不同的irqchip，在不同架构编译会通过不同架构设置导出不同的方法，比如risc-v的aia以及plic会有单独的host_plic其中在没有aia的情况下还会多导出vplic_global_emul_handler, vplic_hart_emul_handler
同样在arm中会多出不同实现的不同方法，比如gicv2和gicv3导出均不同
不过都会导出以下函数
inject_irq, percpu_init, primary_init_early, primary_init_late

uart
在这一层同样的hvisor封装了console_putchar、console_getchar

```

AI生成：

## Entry 启动流程

```rust
arch::trap::install_trap_vector();
ArchCpu;
cpu_start(cpu_id, arch_entry as _, host_dtb);
```

- arch_entry：架构特定入口
- cpu_start：启动目标 CPU
- install_trap_vector：安装异常向量

## IPI 发送机制

```rust
arch_send_event(cpu_id: u64, _sgi_num: u64);
```

- 架构相关 IPI 实现

## MM 子系统

### 初始化方法

- init_hv_page_table(fdt: &fdt::Fdt)
- new_s2_memory_set() -> MemorySet<Stage2PageTable>

### 初始化步骤

- pt_init：页表初始化
- mmio_init：MMIO 区域注册
- irq_bitmap_init：中断位图资源初始化

## Paging 通用抽象接口

### 基础类型定义

```rust
enum PageSize {
    Size4K = 0x1000,
    Size2M = 0x20_0000,
    Size1G = 0x4000_0000,
}

struct Page<VA> {
    vaddr: VA,
    size: PageSize,
}
```

### 通用 PTE Trait（未统一抽象）

```rust
trait GenericPTE {
    fn addr(&self) -> PhysAddr;
    fn flags(&self) -> MemFlags;
    fn is_unused(&self) -> bool;
    fn is_present(&self) -> bool;
    fn is_huge(&self) -> bool;
    fn set_addr(&mut self, paddr: PhysAddr);
    fn set_flags(&mut self, flags: MemFlags, is_huge: bool);
    fn set_table(&mut self, paddr: PhysAddr);
    fn clear(&mut self);
}
```

### Paging 控制指令接口

```rust
trait PagingInstr {
    unsafe fn activate(root_paddr: PhysAddr);
    fn flush(vaddr: Option<usize>);
}
```

### Page Table 抽象接口

```rust
trait GenericPageTableImmut {
    type VA: From<usize> + Into<usize> + Copy;

    unsafe fn from_root(root_paddr: PhysAddr) -> Self;
    fn root_paddr(&self) -> PhysAddr;
    fn query(&self, vaddr: Self::VA) -> PagingResult<(PhysAddr, MemFlags, PageSize)>;
}

trait GenericPageTable: GenericPageTableImmut {
    fn new() -> Self;
    fn map(&mut self, region: &MemoryRegion<Self::VA>) -> HvResult;
    fn unmap(&mut self, region: &MemoryRegion<Self::VA>) -> HvResult;
    fn update(&mut self, vaddr: Self::VA, paddr: PhysAddr, flags: MemFlags) -> PagingResult<PageSize>;
    fn clone(&self) -> Self;
    unsafe fn activate(&self);
    fn flush(&self, vaddr: Option<Self::VA>);
}
```

### 错误类型

```rust
enum PagingError {
    NoMemory,
    NotMapped,
    AlreadyMapped,
    MappedToHugePage,
}

type PagingResult<T> = Result<T, PagingError>;
```

## 不同架构页表实现

### LoongArch 架构

```rust
struct Level4PageTable<VA, PTE, I> {
    inner: Level4PageTableUnlocked<VA, PTE, I>,
    clonee_lock: Arc<Mutex<()>>,
}

type Stage1PageTable = Level4PageTable<GuestPhysAddr, PageTableEntry, S1PTInstr>;
type Stage2PageTable = Level4PageTable<GuestPhysAddr, PageTableEntry, S2PTInstr>;
```

### RISC-V 架构

```rust
struct Level3PageTable<VA, PTE, I> {
    inner: Level3PageTableUnlocked<VA, PTE, I>,
    clonee_lock: Arc<Mutex<()>>,
}

type Stage1PageTable = Level3PageTable<HostPhysAddr, PageTableEntry, S1PTInstr>;
type Stage2PageTable = Level3PageTable<HostPhysAddr, PageTableEntry, S2PTInstr>;
```

## Trap 子系统

- install_trap_vector：安装异常向量
- _hyp_trap_vector：Hypervisor 异常向量（汇编入口）
- _hyp_trap_return：异常返回入口

## IRQ 中断控制

### 通用导出接口

- inject_irq
- percpu_init
- primary_init_early
- primary_init_late

### RISC-V 实现

- 若支持 AIA：走 aia_*
- 否则使用：
    - host_plic
    - vplic_global_emul_handler
    - vplic_hart_emul_handler

### ARM 实现

- 根据 GIC 类型不同导出：
    - GICv2
    - GICv3

## UART 控制台输出

    fn console_putchar(ch: u8);
    fn console_getchar() -> u8;

- 所有架构统一接口，底层访问 UART 寄存器

## 模块

| 模块  | 说明                           |
| ----- | ------------------------------ |
| entry | Hypervisor 启动入口            |
| trap  | 安装异常向量，处理中断/异常    |
| mm    | 页表管理、地址映射、内存初始化 |
| irq   | 中断控制器初始化与中断注入机制 |
| uart  | 控制台串口输入输出接口         |
