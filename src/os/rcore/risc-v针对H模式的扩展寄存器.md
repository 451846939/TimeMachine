# risv-v针对H模式扩展的寄存器

## 新增的CSR


| **CSR 名称** | **地址** | **说明**                                                |
| ------------ | -------- | ------------------------------------------------------- |
| hstatus      | 0x600    | Hypervisor 状态寄存器，类似于 sstatus，保存 H-mode 状态 |
| hedeleg      | 0x602    | Trap 异常委托寄存器                                     |
| hideleg      | 0x603    | 中断委托寄存器                                          |
| hgatp        | 0x680    | Guest 地址翻译与保护寄存器（用于二级页表）              |
| htval        | 0x643    | Guest 虚拟地址（用于异常恢复）                          |
| htinst       | 0x64A    | Guest 引发异常的指令                                    |
| vsstatus     | 0x200    | 虚拟 supervisor 状态寄存器（guest 的 sstatus）          |
| vsie         | 0x204    | 虚拟中断使能（guest 的 sie）                            |
| vstvec       | 0x205    | 虚拟 trap 向量表（guest 的 stvec）                      |
| vsscratch    | 0x240    | 虚拟 scratch 寄存器                                     |
| vsepc        | 0x241    | 虚拟异常程序计数器                                      |
| vscause      | 0x242    | 虚拟 trap 原因                                          |
| vstval       | 0x243    | 虚拟 trap 附加值                                        |
| vsip         | 0x244    | 虚拟中断挂起位                                          |
| vsatp        | 0x280    | 虚拟地址翻译与保护寄存器（guest 的一级页表）            |



## 原有的CSR中新增字段的寄存器

| **原 CSR**                           | **新增字段**              | **说明**                                                     |
| ------------------------------------ | ------------------------- | ------------------------------------------------------------ |
| mstatus                              | TVM, TW, TSR, VS（两位）  | 控制 S-mode 和 VS-mode 的行为，新增 VS 字段用于表示当前虚拟模式 |
| medeleg / mideleg                    | 对虚拟化 Trap 的委托支持  | 允许 M-mode 委托异常给 HS/VS                                 |
| satp                                 | 未改动                    | S 模式页表，H 模式使用 hgatp                                 |
| sstatus                              | VS 模式下表现不同         | 在进入 VS 模式时，其内容由 vsstatus 替代                     |
| scause, sepc, stval, stvec, sscratch | 用于 host supervisor 使用 | 虚拟模式下，guest 使用 vs* 替代这些 CSR                      |



## 如何判断当前是否处于VS-mode

利用 mstatus.MPP 和 hstatus.SPV：

| **CSR** | **字段**         | **含义**                                                     |
| ------- | ---------------- | ------------------------------------------------------------ |
| mstatus | MPP (bits 12:11) | 表示下一次 mret 将跳转的特权模式（00=U, 01=S, 11=M）         |
| hstatus | SPV (bit 8)      | Supervisor Previous Virtual，指明目标是否为 **虚拟 S-mode（VS-mode）** |

| Virtualization Mode (V) | Nominal Privilege | Abbreviation | Name                                | Two-Stage Translation |
| ----------------------- | ----------------- | ------------ | ----------------------------------- | --------------------- |
| 0                       | U                 | U-mode       | User mode                           | Off                   |
| 0                       | S                 | HS-mode      | Hypervisor-extended supervisor mode | Off                   |
| 0                       | M                 | M-mode       | Machine mode                        | Off                   |
| 1                       | U                 | VU-mode      | Virtual user mode                   | On                    |
| 1                       | S                 | VS-mode      | Virtual supervisor mode             | On                    |



## vmm需要注意的寄存器

| 功能类别           | 功能描述                              | CSR 名称            | Guest 可访问 | 说明                              |
| ------------------ | ------------------------------------- | ------------------- | ------------ | --------------------------------- |
| 1. Guest 页表基址  | 指定 Guest 页表根地址（G-stage 页表） | hgatp               | 否           | G-stage 页表，控制二级地址转换    |
| 2. Guest 页表配置  | 指定 Guest 页表格式                   | vsatp               | 是           | VS-mode 可访问，代表 Guest 页表根 |
| 3. Guest 状态控制  | Guest sstatus（虚拟）                 | vsstatus            | 是           | Guest 虚拟视角下的 sstatus        |
|                    | Guest stvec                           | vstvec              | 是           | Guest Trap 入口                   |
|                    | Guest sie                             | vsie                | 是           | Guest 中断使能                    |
|                    | Guest scounteren                      | vscounteren         | 是           | Guest 可访问计数器标志位          |
|                    | Guest sscratch                        | vsscratch           | 是           | Guest 临时寄存器                  |
|                    | Guest sepc                            | vsepc               | 是           | Guest 异常返回地址                |
|                    | Guest scause                          | vscause             | 是           | Guest trap 原因                   |
|                    | Guest stval                           | vstval              | 是           | trap 相关值                       |
|                    | Guest sip                             | vsip                | 是           | Guest 中断挂起                    |
| 4. VMM Trap 控制   | Trap 转发及来源标识                   | hstatus             | 否           | 控制 trap 是否来自 VS-mode        |
|                    | HS-mode Trap 入口地址                 | htvec               | 否           | Hypervisor trap vector            |
|                    | 保存 Guest trap 原因                  | scause              | 否           | HS-mode 捕获 trap 后读取          |
|                    | 保存 Guest 异常值                     | stval               | 否           |                                   |
|                    | Guest Trap 的指令信息                 | htinst              | 否           |                                   |
|                    | Guest trap 地址相关信息               | htval               | 否           |                                   |
| 5. Guest 切换控制  | trap 来源与 guest 返回                | hstatus.SPVP, SPV   | 否           | 判断当前 trap 来自 VS-mode        |
|                    | G-stage 地址翻译控制                  | hgatp, hstatus.VTVM | 否           |                                   |
| 6. 计数器权限控制  | 允许 guest 访问计数器                 | hcounteren          | 否           |                                   |
|                    | Host 控制自己的计数器访问             | scounteren          | 否           |                                   |
| 7. 虚拟中断控制    | Guest 中断挂起                        | hvip                | 否           | 设置 guest 的 sip                 |
|                    | Guest 中断使能                        | hie                 | 否           | 控制是否启用虚拟中断转发          |
| 8. Guest XLEN 设置 | 控制 guest XLEN                       | vsstatus.VSXL       | 是           | 设置 Guest 的 XLEN                |
| 9. trap 返回与指令 | trap 返回指令                         | sret, mret          | 是（特殊）   | 控制从异常返回到 VS-mode          |
|                    | 启用虚拟化扩展标志                    | misa (H bit)        | 否           | 表示 H 扩展支持                   |
