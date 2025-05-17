# axvisor 运行流程

## 1._start()

在`scripts/lds/linker.lds.S`中我们可以看到`ENTRY(_start)`，`axvisor`其实是复用了`arceos`的组件我们在`Cargo.toml`也可以看到相应的依赖

```toml
axstd = { git = "https://github.com/arceos-hypervisor/arceos.git", branch = "vmm", features = [
    "alloc",
    "paging",
    # "fs",
    "irq",
    "hv",
    "multitask",
    # "sched_rr"
]}
```

找到了我们的入口那我们可以直接跟随`_start`进入我们的vmm最开始的启动流程这里会根据不同的平台架构调用不同的`_start`具体的定义在arceos项目中的`modules/platform/xxxx/boot.rs`中定义的`_start`函数，这里以**qemu**运行**risc-v**来举例子，会调用我们的`modules/platform/riscv64_qemu_virt/boot.rs`中的`_start`

```rust
unsafe extern "C" fn _start() -> ! {
    // PC = 0x8020_0000
    // a0 = hartid
    // a1 = dtb
    core::arch::naked_asm!("
        mv      s0, a0                  // save hartid
        mv      s1, a1                  // save DTB pointer
        la      sp, {boot_stack}
        li      t0, {boot_stack_size}
        add     sp, sp, t0              // setup boot stack

        call    {init_boot_page_table}
        call    {init_mmu}              // setup boot page table and enabel MMU

        li      s2, {phys_virt_offset}  // fix up virtual high address
        add     sp, sp, s2

        mv      a0, s0
        mv      a1, s1
        la      a2, {entry}
        add     a2, a2, s2
        jalr    a2                      // call rust_entry(hartid, dtb)
        j       .",
        phys_virt_offset = const PHYS_VIRT_OFFSET,
        boot_stack_size = const TASK_STACK_SIZE,
        boot_stack = sym BOOT_STACK,
        init_boot_page_table = sym init_boot_page_table,
        init_mmu = sym init_mmu,
        entry = sym super::rust_entry,
    )
}
```

这段代码主要做的事：

1. **保存参数**：

   - 启动 hart 时，OpenSBI 通过 a0/a1 传 hartid 和 dtb。

   - 为了后续使用，需要将它们存入 s0 和 s1。

2. **设置栈**：

   - 启动时没有栈，必须自己设置一个临时栈空间（通常在 .bss 段中预分配）。

3. **开启 MMU**：

   - 初始化页表 → 设置页表基地址 → 开启 MMU。

4. **地址修正**：

   - 启动时代码运行在物理地址，启用 MMU 后访问地址需要加上偏移量 PHYS_VIRT_OFFSET。

5. **进入内核主函数**：

   - 最终将 hartid 和 dtb 指针传给 Rust 层的 rust_entry，开始真正的内核执行逻辑。

之后会进入我们同样定义在`boot.rs`中的`rust_entry`，该函数用于在 MMU 启动后完成**早期初始化流程**，准备好执行真正的 `rust_main`

## 2. rust_main

到了这里就和我们的arceos的启动流程一致

```rust
pub extern "C" fn rust_main(cpu_id: usize, dtb: usize) -> ! {
		......
    // 初始化内存管理与分配器
    #[cfg(feature = "alloc")]
    init_allocator();
    #[cfg(feature = "paging")]
    axmm::init_memory_management();
    // 初始化平台设备
    axhal::platform_init();
    // 初始化调度器
    #[cfg(feature = "multitask")]
    axtask::init_scheduler();
		....
    // 初始化中断
    #[cfg(feature = "irq")]
    init_interrupt();
		.....
    // 调用静态构造函数（全局初始化器）
    ctor_bare::call_ctors();
    // 启动主程序
    unsafe { main() };
    // 主程序退出逻辑
    #[cfg(feature = "multitask")]
    axtask::exit(0);
		....
}
```

这里删除了部分没有用到的feature相关的代码比如smp和fs等可以看到主要复用了内存管理、调度、中断处理的部分以及不同平台的初始化的部分最终来到我们的`main` 函数，至此终于调用到我们的`axvisor`部分了

## 3.axvisor main

```rust
#[unsafe(no_mangle)]
fn main() {
    logo::print_logo();
    info!("Starting virtualization...");

    // 检查并启用虚拟化扩展
    info!("Hardware support: {:?}", axvm::has_hardware_support());
    hal::enable_virtualization();

    // 启动虚拟机管理器
    vmm::init();
    vmm::start();

    info!("VMM shutdown");
}
```

可以看到这个函数非常简洁检查硬件是否支持，之后开启虚拟化

还记得我们没有复用arceos的smp功能吗，在`enable_virtualization`函数中会补上这一块

```rust
pub(crate) fn enable_virtualization() {
    static CORES: AtomicUsize = AtomicUsize::new(0);
    for cpu_id in 0..config::SMP {
        thread::spawn(move || {
            // 绑定线程到目标 CPU
            ax_set_current_affinity(AxCpuMask::one_shot(cpu_id)).expect("Set affinity failed");
            vmm::init_timer_percpu();

            let percpu = unsafe { AXVM_PER_CPU.current_ref_mut_raw() };
            percpu.init(this_cpu_id()).expect("init percpu failed");
            percpu.hardware_enable().expect("enable virtualization failed");
            info!("Virtualization enabled on core {}", cpu_id);
            CORES.fetch_add(1, Ordering::Release);
        });
    }
    // 等待所有核完成初始化
    while CORES.load(Ordering::Acquire) != config::SMP {
        thread::yield_now();
    }
}
```

这一段的做的事大概如下

1. **遍历所有 CPU 核心**

   使用 for cpu_id in 0..config::SMP 遍历所有核，为每个核心创建一个新线程。

2. **绑定线程到目标 CPU**

   在每个线程中，通过 ax_set_current_affinity 把线程绑定到对应的 CPU（确保后续虚拟化操作在该核上执行）。

3. **初始化每核定时器**

   调用 vmm::init_timer_percpu() 设置当前核的虚拟化相关定时器或中断源。

4. **获取并初始化 percpu 状态结构**

   使用 AXVM_PER_CPU.current_ref_mut_raw() 获取当前核的 per-CPU 虚拟化状态结构，并调用 init() 进行初始化。

5. **开启当前核的虚拟化支持**

   调用 percpu.hardware_enable() 启动该核的硬件虚拟化扩展（如设置 CSR 或控制寄存器等）。

6. **记录已完成的核数**

   使用 CORES.fetch_add(1, Ordering::Release) 原子计数器记录已完成初始化的 CPU 核数量。

7. **等待所有 CPU 核完成初始化**

   主线程中通过 while CORES.load(Ordering::Acquire) != config::SMP 循环等待所有核准备就绪，期间通过 thread::yield_now() 避免忙等。

