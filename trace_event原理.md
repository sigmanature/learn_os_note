 好的，这个问题问得非常好，直击了 Linux 内核事件跟踪（Tracing）框架的核心设计。理解了 `TRACE_EVENT` 宏的实现，就基本理解了 Ftrace Event 子系统是如何工作的。

我们来一步步“扒开”这个宏，看看它背后到底隐藏了什么。

### 总结：我们日常使用的是哪种定义？

答案是：**在不同的编译阶段，这三种定义都被用到了。**

这不是一个简单的 `#ifdef` 选择，而是一种非常精妙的 C 预处理器技巧，称为 **“多重包含” (Multi-pass Inclusion)**。同一个头文件（比如 `include/trace/events/sched.h`）会被多次包含，每次包含之前，`TRACE_EVENT` 宏都被定义成不同的样子，从而从同一份源代码中生成出完全不同的 C 代码。

1.  **日常调用代码 (The Caller):** 当你在内核其他地方写 `trace_sched_switch(...)` 时，你使用的是 `linux/tracepoint.h` 中的定义。这会生成一个轻量级的内联函数，它通过静态分支（static branch）来检查 tracepoint 是否启用，如果未启用，开销几乎为零。
2.  **定义 Tracepoint 的源文件 (The Owner):** 在某个特定的 `.c` 文件中（例如 `kernel/sched/core.c`），会定义 `CREATE_TRACE_POINTS`。此时，`trace/define_trace.h` 中的定义生效。这会生成 tracepoint 的核心数据结构 (`struct tracepoint`) 和探针迭代函数。
3.  **Ftrace 核心框架 (The Tracing Framework):** 在编译 `kernel/trace/trace_events.c` 时，它会包含 `trace/trace_events.h`。这个头文件会用一种更复杂的方式，分多个“阶段”（Stage 1-7）反复包含所有事件头文件。每个阶段都重新定义 `TRACE_EVENT`，用于生成 Ftrace 所需的各种元数据结构，比如事件的二进制格式、打印函数、字段信息等。

下面我们来详细分解这个过程。

---

### 第一层：`linux/tracepoint.h` - 为调用者服务

这是最外层，也是最常见的一层。当内核代码仅仅是想 *触发* 一个已经存在的 trace event 时，它包含的头文件（如 `<trace/events/sched.h>`）最终会依赖 `linux/tracepoint.h`。

在这种情况下，`CREATE_TRACE_POINTS` **没有** 被定义。我们来看看相关的宏展开：

1.  **`TRACE_EVENT(name, proto, args, ...)`**
    这个宏在 `linux/tracepoint.h` 中被定义为：
    ```c
    #define TRACE_EVENT(name, proto, args, struct, assign, print)	\
        DECLARE_TRACE_EVENT(name, PARAMS(proto), PARAMS(args))
    ```

2.  **`DECLARE_TRACE_EVENT(name, proto, args)`**
    接着展开为：
    ```c
    #define DECLARE_TRACE_EVENT(name, proto, args)				\
        __DECLARE_TRACE(name, PARAMS(proto), PARAMS(args),		\
                cpu_online(raw_smp_processor_id()),		\
                PARAMS(void *__data, proto))
    ```

3.  **`__DECLARE_TRACE(name, proto, args, cond, data_proto)`**
    这是最核心的定义，它生成了我们平时调用的 `trace_...()` 函数：
    ```c
    #define __DECLARE_TRACE(name, proto, args, cond, data_proto)		\
        // ... (省略了注册/反注册的辅助函数) ...
        static inline bool						\
        trace_##name##_enabled(void)					\
        {								\
            return static_branch_unlikely(&__tracepoint_##name.key);\
        }								\
        /* ... */
        static inline void trace_##name(proto)				\
        {								\
            if (static_branch_unlikely(&__tracepoint_##name.key))	\
                __do_trace_##name(args);			\
            /* ... (RCU 锁调试检查) ... */		\
        }
    ```

**这一层的作用：**

*   为每个 trace event 创建一个 `static inline` 函数，例如 `trace_sched_switch()`。
*   这个函数内部使用了一个 `static_branch_unlikely`（静态分支）。这是内核的一项性能优化：
    *   如果这个 tracepoint 没有被任何工具（如 Ftrace, perf）启用，`__tracepoint_##name.key` 为 false。编译器会直接跳过 `if` 块内的代码，几乎没有运行时开销。
    *   如果被启用，才会执行 `__do_trace_##name()`，这个函数会真正地去调用注册的探针（probe）。
*   它还提供了 `trace_sched_switch_enabled()` 函数来检查该事件是否启用。

**结论：** 当你的代码只是 `#include <trace/events/foo.h>` 并调用 `trace_foo()` 时，你使用的是这一套定义。它的目标是 **高性能、低开销**。

---

### 第二层：`trace/define_trace.h` - 为定义者服务

每个 tracepoint 必须在内核的某个地方被真正地“实体化”。这是通过在一个 `.c` 文件中定义 `CREATE_TRACE_POINTS` 然后再包含对应的事件头文件来实现的。

当 `CREATE_TRACE_POINTS` 被定义时，`trace/define_trace.h` 中的宏会生效。

1.  **`TRACE_EVENT(name, proto, args, ...)`**
    这次它被定义为：
    ```c
    #define TRACE_EVENT(name, proto, args, tstruct, assign, print)	\
        DEFINE_TRACE(name, PARAMS(proto), PARAMS(args))
    ```

2.  **`DEFINE_TRACE(name, proto, args)`**
    展开为 `__DEFINE_TRACE_EXT`：
    ```c
    #define DEFINE_TRACE(_name, _proto, _args)				\
        __DEFINE_TRACE_EXT(_name, NULL, PARAMS(_proto), PARAMS(_args));
    ```

3.  **`__DEFINE_TRACE_EXT(...)`**
    这个宏负责生成 tracepoint 的所有核心数据结构：
    ```c
    #define __DEFINE_TRACE_EXT(_name, _ext, proto, args)			\
        /* 字符串表，保存名字 "name" */ \
        static const char __tpstrtab_##_name[]				\
        __section("__tracepoints_strings") = #_name;			\
        /* ... 其他符号声明 ... */ \
        /* 核心数据结构！*/ \
        struct tracepoint __tracepoint_##_name	__used			\
        __section("__tracepoints") = {					\
            .name = __tpstrtab_##_name,				\
            .key = STATIC_KEY_FALSE_INIT, /* 静态分支的 key */ \
            /* ... 其他字段初始化 ... */ \
        };								\
        /* 指向该结构体的指针，用于内核遍历所有 tracepoints */ \
        __TRACEPOINT_ENTRY(_name);					\
        /* 迭代器函数，当 tracepoint 被触发时，它会遍历并调用所有探针 */ \
        int __traceiter_##_name(void *__data, proto)			\
        {								\
            /* ... 循环调用注册的探针 ... */ \
        }								\
        /* ... */
    ```

**这一层的作用：**

*   定义一个 `struct tracepoint` 类型的全局变量，如 `__tracepoint_sched_switch`。
*   将这个结构体放入一个特殊的 ELF section `.tracepoints` 中。内核启动时可以扫描这个 section 来找到系统中所有的 tracepoints。
*   定义 `__traceiter_##name()` 函数，这是实际执行探针逻辑的地方。
*   把 `name` 字符串和指向 `struct tracepoint` 的指针也放入特殊的 sections，方便查找。

**结论：** 这一层是 tracepoint 的“根”。它创建了 tracepoint 的实例，使得探针可以注册到它上面。这个过程只会在一个编译单元（一个 `.c` 文件）中发生一次。

---

### 第三层：`trace/trace_events.h` - 为 Ftrace 框架服务

这是最复杂的一层，它专门为 Ftrace 的事件跟踪系统服务。`kernel/trace/trace_events.c` 会包含这个头文件，它会扫描并处理所有的 `trace/events/*.h` 文件。

`trace/trace_events.h` 文件本身就是一个“驱动程序”，它会分阶段（stage）多次包含用户定义的事件头文件（如 `sched.h`）。在每个阶段开始前，它都会重新定义 `TRACE_EVENT` 和其他相关宏。

我们简单看几个关键阶段：

*   **Stage 1: 定义二进制数据结构**
    *   `TRACE_EVENT` 被重定义，最终展开 `DECLARE_EVENT_CLASS`。
    *   `DECLARE_EVENT_CLASS` 被定义为创建一个 `struct trace_event_raw_##name`。
    *   你定义的 `TP_STRUCT__entry(...)` 宏中的 `__field` 和 `__array` 会在这个结构体中展开成真正的成员变量。
    *   **作用：** 定义了当这个事件发生时，写入 ring buffer 的数据的二进制布局。

    ```c
    // Stage 1 的宏定义
    #define DECLARE_EVENT_CLASS(name, proto, args, tstruct, assign, print)	\
        struct trace_event_raw_##name {					\
            struct trace_entry	ent;				\
            tstruct  /* TP_STRUCT__entry(...) 的内容在这里展开 */ \
            char			__data[];			\
        };
    ```

*   **Stage 3: 定义文本打印函数**
    *   `DECLARE_EVENT_CLASS` 再次被重定义，这次是用来创建一个 `trace_raw_output_##call` 函数。
    *   这个函数的主体逻辑会使用你在 `TP_printk(...)` 中定义的格式化字符串。
    *   **作用：** 定义了如何将 Stage 1 中保存的二进制数据转换成人类可读的文本格式（就是你在 `cat /sys/kernel/debug/tracing/trace` 中看到的格式）。

*   **Stage 4-7: 定义元数据和回调函数**
    *   这些阶段会定义 `trace_event_class`、`trace_event_call` 等 Ftrace 内部管理结构。
    *   它会定义一个 `trace_event_raw_event_##call` 函数，这个函数就是 Ftrace 注册到 tracepoint 上的探针（probe）。
    *   当 `trace_sched_switch()` 被调用时，`__traceiter_sched_switch` 会被执行，它会调用 `trace_event_raw_event_sched_switch`。
    *   这个函数会从 `TP_fast_assign` 宏中获取数据，填充 Stage 1 定义的 `trace_event_raw_` 结构体，然后把它写入 ring buffer。

**这一层的作用：**

*   为 Ftrace 框架自动生成所有必要的代码和数据结构。
*   解析 `TP_STRUCT__entry` 来创建二进制格式。
*   解析 `TP_fast_assign` 来创建数据填充逻辑。
*   解析 `TP_printk` 来创建文本输出函数。
*   将所有东西粘合在一起，创建一个 Ftrace 可以理解和管理的 "event"。

### 最终流程串讲

1.  在 `kernel/sched/core.c` 中，`#define CREATE_TRACE_POINTS` 导致 `trace/define_trace.h` 生效，创建了 `__tracepoint_sched_switch` 这个核心结构。
2.  在编译 `kernel/trace/trace_events.c` 时，`trace/trace_events.h` 通过多阶段处理 `trace/events/sched.h`，创建了 Ftrace 所需的 `trace_event_raw_sched_switch` 结构体、`trace_raw_output_sched_switch` 打印函数以及 `trace_event_raw_event_sched_switch` 回调探针，并将这个探针注册到 `__tracepoint_sched_switch` 上。
3.  在内核的任何其他地方，当 `schedule()` 函数被调用，它内部的 `trace_sched_switch()` 被触发。由于 `linux/tracepoint.h` 的定义，这会变成一个 `static_branch` 检查。
4.  假设 Ftrace 已经启用了 `sched_switch` 事件，`static_branch` 判断为真，执行 `__do_trace_sched_switch()`，最终调用 `__traceiter_sched_switch()`。
5.  `__traceiter_sched_switch()` 遍历探针列表，调用了 Ftrace 注册的 `trace_event_raw_event_sched_switch` 探针。
6.  这个探针执行 `TP_fast_assign` 的逻辑，把 `prev` 和 `next` 任务的信息填入 `trace_event_raw_sched_switch` 结构体，并写入 ring buffer。
7.  当用户 `cat` trace 文件时，Ftrace 从 ring buffer 读取二进制数据，并调用 `trace_raw_output_sched_switch` 函数，使用 `TP_printk` 的格式将其打印出来。

这个设计虽然复杂，但实现了极高的灵活性和性能。开发者只需要用 `TRACE_EVENT` 定义一次事件，所有相关的代码（调用点、定义、Ftrace 框架）都能自动生成，并且性能开销在事件关闭时几乎为零。

