
# 任务调度 (axtask)

<cite>
**本文档中引用的文件**   
- [lib.rs](file://modules/axtask/src/lib.rs)
- [api.rs](file://modules/axtask/src/api.rs)
- [task.rs](file://modules/axtask/src/task.rs)
- [run_queue.rs](file://modules/axtask/src/run_queue.rs)
- [timers.rs](file://modules/axtask/src/timers.rs)
- [wait_queue.rs](file://modules/axtask/src/wait_queue.rs)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概述](#架构概述)
5. [详细组件分析](#详细组件分析)
6. [依赖分析](#依赖分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)

## 简介
`axtask` 模块是 ArceOS 操作系统中的核心任务管理组件，负责实现多任务环境下的任务创建、调度、同步和生命周期管理。该模块支持多种可配置的调度算法（FIFO、RR、CFS），并为单核与多核（SMP）环境提供了完整的任务控制机制。其设计目标是在保证实时性的同时，提供公平且高效的 CPU 资源分配策略。

## 项目结构
`axtask` 模块位于 `modules/axtask` 目录下，是 ArceOS 内核的关键组成部分。它通过清晰的文件划分实现了高内聚、低耦合的设计。

```mermaid
graph TD
A[axtask模块] --> B[lib.rs]
A --> C[api.rs]
A --> D[task.rs]
A --> E[run_queue.rs]
A --> F[timers.rs]
A --> G[wait_queue.rs]
A --> H[task_ext.rs]
B --> |模块入口与功能开关| C
B --> |条件编译| D
C --> |对外API| D
C --> |调度器选择| E
D --> |任务控制块TCB| E
E --> |就绪队列管理| F
E --> |时间片与抢占| G
F --> |定时唤醒| G
```

**Diagram sources**
- [lib.rs](file://modules/axtask/src/lib.rs)
- [api.rs](file://modules/axtask/src/api.rs)
- [task.rs](file://modules/axtask/src/task.rs)
- [run_queue.rs](file://modules/axtask/src/run_queue.rs)
- [timers.rs](file://modules/axtask/src/timers.rs)
- [wait_queue.rs](file://modules/axtask/src/wait_queue.rs)

**Section sources**
- [lib.rs](file://modules/axtask/src/lib.rs)
- [api.rs](file://modules/axtask/src/api.rs)

## 核心组件

`axtask` 模块的核心由任务控制块（TCB）、运行队列（Run Queue）、调度器（Scheduler）和等待队列（Wait Queue）构成。这些组件协同工作，实现了从任务创建到销毁的完整生命周期管理。任务的状态转换由 `TaskState` 枚举精确描述，并通过原子操作保证了在并发环境下的安全性。

**Section sources**
- [task.rs](file://modules/axtask/src/task.rs#L44-L91)
- [run_queue.rs](file://modules/axtask/src/run_queue.rs#L180-L219)

## 架构概述

`axtask` 模块采用分层架构，将任务抽象、调度逻辑和底层硬件交互分离。其核心是一个可插拔的调度器框架，允许在编译时通过 Cargo 特性（feature）选择不同的调度算法。

```mermaid
graph TB
subgraph "用户空间"
App["应用程序"]
end
subgraph "内核空间"
API["API接口<br>spawn, yield_now, sleep"]
Task["任务控制块(TCB)<br>TaskInner"]
RunQueue["运行队列(Run Queue)<br>AxRunQueue"]
Scheduler["调度器(Scheduler)<br>Fifo/RR/CFS"]
WaitQueue["等待队列(Wait Queue)"]
Timers["定时器(Timers)<br>TimerList"]
end
App --> API
API --> Task
API --> RunQueue
Task --> RunQueue
RunQueue --> Scheduler
RunQueue --> Timers
RunQueue --> WaitQueue
Timers --> RunQueue
WaitQueue --> RunQueue
```

**Diagram sources**
- [api.rs](file://modules/axtask/src/api.rs)
- [task.rs](file://modules/axtask/src/task.rs)
- [run_queue.rs](file://modules/axtask/src/run_queue.rs)

## 详细组件分析

### 任务控制块(TCB)分析
`TaskInner` 结构体是任务控制块（TCB）的具体实现，封装了任务的所有元数据和状态信息。

#### TCB数据结构
```mermaid
classDiagram
class TaskInner {
+id : TaskId
+name : String
+is_idle : bool
+is_init : bool
+entry : Option<*mut dyn FnOnce()>
+state : AtomicU8
+cpumask : SpinNoIrq<AxCpuMask>
+in_wait_queue : AtomicBool
+cpu_id : AtomicU32
+on_cpu : AtomicBool
+timer_ticket_id : AtomicU64
+need_resched : AtomicBool
+preempt_disable_count : AtomicUsize
+exit_code : AtomicI32
+wait_for_exit : WaitQueue
+kstack : Option<TaskStack>
+ctx : UnsafeCell<TaskContext>
+task_ext : AxTaskExt
+tls : TlsArea
+new(entry, name, stack_size) AxTaskRef
+id() TaskId
+name() &str
+state() TaskState
+set_state(state)
+kernel_stack_top() Option<VirtAddr>
+cpu_id() u32
+cpumask() AxCpuMask
+init_task_ext(data) Option<&T>
}
class TaskState {
<<enumeration>>
Running = 1
Ready = 2
Blocked = 3
Exited = 4
}
TaskInner --> TaskState : state
```

**Diagram sources**
- [task.rs](file://modules/axtask/src/task.rs#L44-L91)

#### 任务状态机
```mermaid
stateDiagram-v2
[*] --> Idle
Idle --> Ready : init_scheduler()
Ready --> Running : schedule()
Running --> Ready : yield_current() 或 时间片用尽
Running --> Blocked : blocked_resched() 或 sleep_until()
Running --> Exited : exit_current()
Blocked --> Ready : unblock_one_task() 或 定时器超时
Exited --> [*] : Drop
```

**Diagram sources**
- [task.rs](file://modules/axtask/src/task.rs#L44-L91)
- [run_queue.rs](file://modules/axtask/src/run_queue.rs#L381-L411)

### 运行队列与调度器分析
运行队列是调度器管理就绪任务的核心数据结构。

#### 就绪队列组织方式
```mermaid
flowchart TD
Start([开始])
--> SelectRQ["select_run_queue(task)"]
--> SMP{"是否启用SMP?"}
SMP --> |否| GlobalRQ["返回全局运行队列"]
SMP --> |是| LoadBalance["根据CPU亲和性与负载均衡选择CPU索引"]
--> GetRQ["get_run_queue(index)"]
--> ReturnRQ["返回特定CPU的运行队列"]
style SelectRQ fill:#f9f,stroke:#333
style GlobalRQ fill:#bbf,stroke:#333,color:#fff
style LoadBalance fill:#f96,stroke:#333
style GetRQ fill:#bbf,stroke:#333,color:#fff
style ReturnRQ fill:#bbf,stroke:#333,color:#fff
```

**Diagram sources**
- [run_queue.rs](file://modules/axtask/src/run_queue.rs#L137-L178)

#### 调度算法实现流程
```mermaid
sequenceDiagram
participant Timer as 定时器中断
participant API as api : : on_timer_tick
participant RQ as CurrentRunQueueRef
participant Sched as Scheduler
participant Task as 当前任务
Timer->>API : IRQ触发
API->>RQ : scheduler_timer_tick()
RQ->>Sched : task_tick(curr)
alt 需要重新调度
Sched-->>RQ : 返回true
RQ->>Task : set_preempt_pending(true)
else 不需要
Sched-->>RQ : 返回false
end
```

**Diagram sources**
- [api.rs](file://modules/axtask/src/api.rs#L88-L129)
- [run_queue.rs](file://modules/axtask/src/run_queue.rs#L270-L293)

### 关键流程分析
#### 任务创建流程
```mermaid
sequenceDiagram
participant User as 用户代码
participant API as api : : spawn
participant Task as TaskInner : : new
participant RQ as select_run_queue
participant Sched as Scheduler
User->>API : spawn(f, name, stack_size)
API->>Task : new(f, name, stack_size)
Task->>Task : 分配内核栈(TaskStack : : alloc)
Task->>Task : 初始化上下文(ctx.init)
Task-->>API : 返回TaskInner
API->>RQ : select_run_queue(&task_ref)
RQ->>Sched : add_task(task_ref)
Sched-->>User : 返回AxTaskRef
```

**Diagram sources**
- [api.rs](file://modules/axtask/src/api.rs#L88-L129)
- [task.rs](file://modules/axtask/src/task.rs#L87-L141)

#### 上下文切换流程
```mermaid
flowchart TD
A["switch_to(prev, next)"] --> B{"prev == next?"}
B --> |是| C[返回]
B --> |否| D["保存prev的上下文<br>save_context(&mut prev.ctx)"]
D --> E["恢复next的上下文<br>restore_context(next.ctx)"]
E --> F["执行上下文切换<br>__switch()"]
F --> G["函数返回到next的上下文"]
```

**Diagram sources**
- [run_queue.rs](file://modules/axtask/src/run_queue.rs#L409-L458)

## 依赖分析

`axtask` 模块与其他内核模块存在紧密的依赖关系，形成了一个完整的系统服务链。

```mermaid
graph LR
    axtask --> axhal["axhal (硬件抽象层)"]
    axtask --> axconfig["axconfig (配置)"]
    axtask --> cpumask["cpumask (CPU掩码)"]
    axtask --> axsched["axsched (调度算法库)"]
    axtask --> kernel_guard["kernel_guard (内核保护)"]
    axtask --> timer_list["timer_list (定时器列表)"]
    axtask --> memory_addr["memory_addr (内存地址)"]
    axtask --> kspin["kspin (自旋锁)"]

    axhal -->|提供| time["时间(wall_time)"]
    axhal -->|提供| per