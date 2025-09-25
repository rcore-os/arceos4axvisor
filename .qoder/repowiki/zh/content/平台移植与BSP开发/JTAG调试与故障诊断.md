# JTAG调试与故障诊断

<cite>
**本文档引用的文件**
- [jtag_debug_in_raspi4.md](file://doc/jtag_debug_in_raspi4.md)
- [platform_raspi4.md](file://doc/platform_raspi4.md)
- [main.rs](file://tools/raspi4/chainloader/src/main.rs)
- [aspace.rs](file://modules/axmm/src/aspace.rs)
- [mutex.rs](file://modules/axsync/src/mutex.rs)
</cite>

## 目录
1. [简介](#简介)
2. [JTAG调试环境搭建](#jtag调试环境搭建)
3. [OpenOCD与GDB远程调试配置](#openocd与gdb远程调试配置)
4. [无操作系统阶段调试技术](#无操作系统阶段调试技术)
5. [典型崩溃场景诊断流程](#典型崩溃场景诊断流程)
6. [调试技巧与最佳实践](#调试技巧与最佳实践)

## 简介
本指南详细介绍了基于JTAG的底层调试技术，适用于Arceos在树莓派4等嵌入式平台上的深度问题排查。通过OpenOCD和GDB远程调试工具，开发者可以在系统启动早期阶段进行单步调试、设置断点、查看寄存器状态和内存内容，从而有效诊断和解决复杂的硬件相关问题。

**Section sources**
- [jtag_debug_in_raspi4.md](file://doc/jtag_debug_in_raspi4.md#L0-L15)
- [platform_raspi4.md](file://doc/platform_raspi4.md#L0-L10)

## JTAG调试环境搭建
### 硬件连接要求
进行JTAG调试需要以下硬件设备：
1. JTAG调试器（如H-JLINK v9）
2. 串口转TTL模块（CH340）
3. 杜邦线
4. 笔记本电脑

### 树莓派4引脚连接
按照以下对应关系连接JTAG调试器与树莓派4：

| GPIO编号 | 信号名称 | JTAG引脚 | 备注 |
|---------|---------|--------|-----|
| - | VTREF | 1 | 连接到3.3V |
| - | GND | 4 | 接地 |
| 22 | TRST | 3 | - |
| 26 | TDI | 5 | - |
| 27 | TMS | 7 | - |
| 25 | TCK | 9 | - |
| 23 | RTCK | 11 | - |
| 24 | TDO | 13 | - |

同时，将CH340模块连接到bcm2711的8、10、12引脚（Rx对Tx，Tx对Rx，GND对GND）。

**Section sources**
- [jtag_debug_in_raspi4.md](file://doc/jtag_debug_in_raspi4.md#L17-L83)

## OpenOCD与GDB远程调试配置
### 调试镜像编译
首先需要编译支持JTAG调试的镜像：

```bash
make A=examples/helloworld MYPLAT=axplat-aarch64-raspi clean
make A=examples/helloworld MYPLAT=axplat-aarch64-raspi JTAG=y
```

此命令会生成一个支持JTAG调试的kernel8.img文件。

### 启动OpenOCD服务
在终端中运行以下命令启动OpenOCD服务：

```bash
make A=examples/helloworld MYPLAT=axplat-aarch64-raspi openocd
```

成功启动后，OpenOCD会监听端口3333用于GDB连接，并显示类似以下信息：
- 检测到JTAG设备：rpi4.tap
- 四个核心均可用，每个核心有6个断点和4个观察点
- GDB服务器已启动并监听3333-3336端口

### GDB调试会话
在另一个终端中启动GDB：

```bash
make A=examples/helloworld MYPLAT=axplat-aarch64-raspi gdb
```

然后执行以下调试命令序列：

```gdb
(gdb) target extended-remote :3333
(gdb) set $pc=0x80000
(gdb) monitor poll
(gdb) break rust_entry
(gdb) b *0x81a90
(gdb) break rust_main
(gdb) b *0x82888
(gdb) delete 1 3
(gdb) continue
```

**Section sources**
- [jtag_debug_in_raspi4.md](file://doc/jtag_debug_in_raspi4.md#L167-L266)
- [main.rs](file://tools/raspi4/chainloader/src/main.rs#L197-L218)

## 无操作系统阶段调试技术
### pre-main阶段调试要点
在无操作系统支持阶段进行调试时，需要注意以下关键点：

1. **程序计数器设置**：由于JTAG调试限制，需要手动设置PC寄存器到正确的加载地址0x80000。
2. **内存映射配置**：修改`/modules/axconfig/src/platform/raspi4_aarch64`中的"kernel-base-vaddr"为"0x8_0000"，"phys-virt-offset"为"0x0"。
3. **单核模式**：设置SMP=1以确保单核运行，避免多核同步问题。

### 调试流程控制
系统在加载后会进入无限循环等待GDB连接，这是通过以下代码实现的：

```rust
#[cfg(feature = "enable_jtag_debug")]
fn print_jtag_info_and_wait_forever() {
    println!("@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@");
    println!("@ You're using a JTAG debug image.  @");
    println!("@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@");
    // ... 显示调试说明 ...
    
    // 等待GDB连接
    cpu::wait_forever()
}
```

这种设计允许开发者在系统启动前建立完整的调试连接。

**Section sources**
- [platform_raspi4.md](file://doc/platform_raspi4.md#L11-L28)
- [main.rs](file://tools/raspi4/chainloader/src/main.rs#L162-L199)

## 典型崩溃场景诊断流程
### 页错误诊断
当发生页错误时，可以通过以下步骤进行诊断：

1. 在GDB中设置页错误异常处理函数断点
2. 触发可能导致页错误的操作
3. 当断点命中时，检查触发页错误的虚拟地址和访问权限
4. 使用`info registers`查看CPU寄存器状态
5. 使用`x/10gx <address>`查看内存内容

页错误处理的核心逻辑位于`aspace.rs`文件中：

```rust
pub fn handle_page_fault(&mut self, vaddr: VirtAddr, access_flags: PageFaultFlags) -> bool {
    if !self.va_range.contains(vaddr) {
        return false;
    }
    if let Some(area) = self.areas.find(vaddr) {
        let orig_flags = area.flags();
        if orig_flags.contains(access_flags) {
            return area
                .backend()
                .handle_page_fault(vaddr, orig_flags, &mut self.pt);
        }
    }
    false
}
```

### 死锁诊断
对于死锁问题，可以采用以下诊断方法：

1. 在互斥锁的关键位置设置断点
2. 观察任务等待队列的状态
3. 检查锁的所有者ID是否合理
4. 分析任务调度顺序

互斥锁的实现包含所有权跟踪机制，有助于诊断死锁：

```rust
impl RawMutex {
    fn lock(&self) {
        let current_id = current().id().as_u64();
        loop {
            match self.owner_id.compare_exchange_weak(
                0,
                current_id,
                Ordering::Acquire,
                Ordering::Relaxed,
            ) {
                Ok(_) => break,
                Err(owner_id) => {
                    assert_ne!(
                        owner_id,
                        current_id,
                        "{} tried to acquire mutex it already owns.",
                        current().id_name()
                    );
                    self.wq.wait_until(|| !self.is_locked());
                }
            }
        }
    }
}
```

**Section sources**
- [aspace.rs](file://modules/axmm/src/aspace.rs#L275-L320)
- [mutex.rs](file://modules/axsync/src/mutex.rs#L0-L150)

## 调试技巧与最佳实践
### 多终端工作流
建议使用多终端工作流进行调试：

1. 终端A：保持miniload运行，负责二进制传输
2. 终端B：运行OpenOCD，管理JTAG连接
3. 终端C：运行GDB，进行交互式调试

推荐使用zellij等终端复用工具来管理多个调试会话。

### 内存检查技巧
- 使用`x/<n><fmt><size> <addr>`命令查看内存内容
- 使用`info proc mappings`查看内存映射
- 使用`monitor poll`命令在MMU启用/禁用间切换

### 断点管理策略
1. 在关键函数入口设置断点（如rust_entry、rust_main）
2. 使用硬件断点而非软件断点以减少性能影响
3. 合理管理断点数量，避免超出硬件限制（树莓派4每个核心支持6个断点）

### 性能考虑
- JTAG调试会显著降低系统性能，仅在必要时使用
- 在完成初步调试后，可移除JTAG依赖以提高运行效率
- 注意调试代码可能引入的时序变化，影响实时性敏感的应用

**Section sources**
- [jtag_debug_in_raspi4.md](file://doc/jtag_debug_in_raspi4.md#L267-L276)