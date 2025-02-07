# 
Linux头文件包含

## 用户:
我对Linux中代码的include现在有些不明白 
不明白的点在你观察图中的do_mounts.h头文件
其包含<linux/kernel.h>头文件 我使用vscode的f12查看定义定位到了linux-6.13.1 文件夹下的include下的linux的kernel.h 呃 我是有点没明白这是怎么做到的
因为do_mounts.h是在init文件夹下的 按理来说其应该#include "../include/linux/types.h" 
可是为什么没有这样呢?

## ChatGPT:
你的困惑在于为什么 `do_mounts.h` 可以直接 `#include <linux/kernel.h>`，而不是使用相对路径 `#include "../include/linux/kernel.h"`。

这是因为 Linux 内核的构建系统在编译时会设置包含路径（include path）。

**编译过程中的包含路径**

在编译大型项目（如 Linux 内核）时，编译器需要知道去哪里寻找头文件。除了在代码中用相对路径或绝对路径指定头文件外，还可以通过设置编译器的“包含路径”选项来告诉编译器头文件的搜索位置。

对于 Linux 内核，构建系统（通常是 Makefiles）会配置一个包含路径列表，其中包含了内核源代码树中所有相关的头文件目录。这个列表通常包括：

*   `include/` 目录：包含内核的公共头文件。
*   `arch/<architecture>/include/` 目录：包含特定体系结构的头文件。
*   以及其他一些目录。

当编译器遇到 `#include <linux/kernel.h>` 这样的语句时，它会依次在这些包含路径中搜索 `linux/kernel.h` 文件。因为`linux-6.13.1/include/linux/` 在编译的include参数中, 所以其可以被寻找到

**VS Code 的智能感知**

VS Code 的智能感知（IntelliSense）功能（包括你使用的 F12 跳转到定义）会尝试模拟编译器的行为。它会解析项目的构建配置（或者根据一些启发式方法猜测），找到头文件的搜索路径，从而正确地解析 `#include` 指令并找到头文件的定义。

**总结**
do_mounts.h include linux/kernel.h的原理:
1.  **构建系统配置**: Linux 内核的构建系统 (Makefiles) 会设置编译器的包含路径, 确定了include的搜索范围。
2.  **编译器搜索**: 编译器根据包含路径来搜索 `#include` 指令中的头文件。
3.  **VS Code 模拟**: VS Code 的智能感知功能会模拟编译器的行为，根据构建配置找到头文件。

所以，即使 `do_mounts.h` 文件位于 `init/` 目录下，它也可以直接 `#include <linux/kernel.h>`，因为构建系统已经将 `include/linux/` 目录添加到了编译器的包含路径中。


> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
好 我现在直接看内核的makefile是看不明白的 
所以说我要向你问些问题 因为我从没学过makefile
首先是你说构建系统拥有设置编译器包含路径选项的能力
那makefile 尤其是顶层的makefile 一般是如何做到这点的? 使用什么样的命令呢? 能产生什么样的效果呢?

## ChatGPT:
好的，我来解释一下 Makefile 如何设置编译器的包含路径，以及它产生什么样的效果。即使你没学过 Makefile，我也会尽量用通俗易懂的方式来说明。

**Makefile 的作用**

首先，要理解 Makefile 的作用。Makefile 就像一个“构建说明书”，它告诉编译器（更准确地说是构建工具 `make`）：

1.  **项目中有哪些文件**：源代码文件、头文件等。
2.  **文件之间的依赖关系**：例如，`main.c` 文件依赖于 `module.h` 头文件。
3.  **如何编译每个文件**：使用什么编译器、编译选项等。
4.  **如何链接生成最终的可执行文件或库**。

**Makefile 如何设置包含路径**

Makefile 通过设置编译器的选项来指定包含路径。对于 C/C++ 编译器（如 GCC），常用的选项是 `-I`（大写字母 i）。

**`-I` 选项**

`-I` 选项后面跟着一个目录路径，告诉编译器在这个目录中搜索头文件。例如：

```makefile
CFLAGS = -I./include -I../common_headers
```

在这个例子中：

*   `CFLAGS` 是一个 Makefile 变量，通常用来存放 C 编译器的选项。
*   `-I./include` 告诉编译器在当前目录下的 `include` 子目录中搜索头文件。
*   `-I../common_headers` 告诉编译器在上一级目录下的 `common_headers` 子目录中搜索头文件。

**在顶层 Makefile 中设置包含路径**

Linux 内核的顶层 Makefile 通常会做以下几件事情来设置包含路径：

1.  **定义变量**：定义一些变量来表示包含路径，例如：

    ```makefile
    export KBUILD_CPPFLAGS := -I$(srctree)/include
    ```
    *   这里，`KBUILD_CPPFLAGS` 是一个变量，用于 C 预处理器（在编译之前处理 `#include` 指令）。
    *   `-I$(srctree)/include` 指定了包含路径为源代码树根目录下的 `include` 目录。`$(srctree)` 是另一个变量，通常指向内核源代码的根目录。
    *   `export` 关键字表示这个变量会传递给子 Makefile。

2.  **包含其他 Makefile**：顶层 Makefile 会包含其他子目录下的 Makefile，这些子 Makefile 可能会添加更多的包含路径。

3.  **传递给编译命令**：在编译具体的源文件时，Makefile 会将 `CFLAGS` 或 `KBUILD_CPPFLAGS` 等变量的值作为参数传递给编译器。例如：

    ```makefile
    $(CC) $(CFLAGS) -c -o myfile.o myfile.c
    ```

    这里，`$(CC)` 是编译器的名称（如 `gcc`），`-c` 表示只编译不链接，`-o myfile.o` 指定输出的目标文件。

**产生的效果**

通过 `-I` 选项设置包含路径，达到的效果是：

1.  **编译器可以找到头文件**：当编译器遇到 `#include <linux/kernel.h>` 这样的语句时，它会在 `-I` 指定的目录中依次查找，直到找到 `linux/kernel.h` 文件。
2.  **简化代码**：源代码中不需要写复杂的相对路径来包含头文件，只需要写 `<header_name.h>` 这样的形式即可。
3.  **模块化**：不同的模块可以将自己的头文件放在不同的目录中，通过 `-I` 选项来管理，避免头文件路径的混乱。

**总结**

*   Makefile 通过编译器的 `-I` 选项来设置包含路径。
*   顶层 Makefile 通常会定义一些变量（如 `CFLAGS`、`KBUILD_CPPFLAGS`）来存储包含路径，并将这些变量传递给编译命令。
*   这样做的效果是让编译器能够找到头文件，简化代码，并实现模块化。

希望这个解释对你有帮助！如果你还有其他关于 Makefile 或 Linux 内核构建的问题，尽管问我。


> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
export srctree := $(if $(KBUILD_EXTMOD),$(abs_srctree),$(srcroot))
这个命令是在进行srctree变量的定义是吗?

## ChatGPT:
是的，这行 Makefile 代码是在定义 `srctree` 变量。我来详细解释一下它的含义：

**分解**

这行代码使用了 Makefile

> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
继续生成

## ChatGPT:
好的，我们继续分解这行 Makefile 代码：

```makefile
export srctree := $(if $(KBUILD_EXTMOD),$(abs_srctree),$(srcroot))
```

这行代码使用了 Makefile 的几个特性：

1.  **`export`**: 这个关键字表示 `srctree` 变量是一个“导出”变量。这意味着它不仅在当前的 Makefile 中有效，还会被传递到所有子 Makefile 中（也就是被这个 Makefile 包含的其他 Makefile）。

2.  **`:=`**: 这是 Makefile 中的“简单赋值”运算符。它表示 `srctree` 变量的值只会在定义时被计算一次，后续使用时不会重新计算。这与 `=`（延迟赋值）不同，`=` 赋值的变量会在每次使用时重新计算。

3.  **`$(if condition,then-part,else-part)`**: 这是 Makefile 的条件函数。它的工作方式如下：
    *   如果 `condition` 不为空（即条件为真），则整个 `$(if ...)` 表达式的值为 `then-part`。
    *   如果 `condition` 为空（即条件为假），则整个 `$(if ...)` 表达式的值为 `else-part`。

4.  **`$(KBUILD_EXTMOD)`**: 这是另一个 Makefile 变量。在 Linux 内核构建系统中，它用来表示是否正在构建一个外部模块（out-of-tree module，例如你自己编写的内核模块）。
    *   如果正在构建外部模块，`$(KBUILD_EXTMOD)` 的值通常不为空。
    *   如果正在构建内核本身，`$(KBUILD_EXTMOD)` 的值通常为空。

5.  **`$(abs_srctree)`**: 这可能是一个 Makefile 变量或函数（具体取决于上下文），用于获取外部模块源代码的绝对路径。

6.  **`$(srcroot)`**: 这是另一个 Makefile 变量，通常指向内核源代码树的根目录。

**整体含义**

将这些部分组合起来，这行代码的含义是：

*   定义一个名为 `srctree` 的导出变量（会传递给子 Makefile）。
*   `srctree` 的值取决于 `$(KBUILD_EXTMOD)` 变量：
    *   如果 `$(KBUILD_EXTMOD)` 不为空（正在构建外部模块），则 `srctree` 的值为 `$(abs_srctree)`（外部模块源代码的绝对路径）。
    *   如果 `$(KBUILD_EXTMOD)` 为空（正在构建内核本身），则 `srctree` 的值为 `$(srcroot)`（内核源代码树的根目录）。

**目的**

这行代码的目的是为了正确地设置源代码树的根目录路径，无论是在构建内核本身还是在构建外部模块时。

*   对于内核本身，`srctree` 应该指向内核源代码的根目录。
*   对于外部模块，`srctree` 应该指向外部模块源代码的根目录。

这样，后续的 Makefile 代码就可以使用 `$(srctree)` 变量来引用源代码文件，而不用关心当前是在构建内核还是外部模块。

**总结**
`export srctree := $(if $(KBUILD_EXTMOD),$(abs_srctree),$(srcroot))`
这行代码通过条件判断，巧妙地处理了内核内模块和内核外模块的不同情况, 保证了`srctree`变量的正确性


> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
我翻到了abs_srctree的定义以及srcroot的定义 以及整段有关有KBUILD_EXTMOD的定义# Use make M=dir or set the environment variable KBUILD_EXTMOD to specify the
# directory of external module to build. Setting M= takes precedence.
ifeq ("$(origin M)", "command line")
  KBUILD_EXTMOD := $(M)
endif

ifeq ("$(origin MO)", "command line")
  KBUILD_EXTMOD_OUTPUT := $(MO)
endif

$(if $(word 2, $(KBUILD_EXTMOD)), \
	$(error building multiple external modules is not supported))

$(foreach x, % :, $(if $(findstring $x, $(KBUILD_EXTMOD)), \
	$(error module directory path cannot contain '$x')))

# Remove trailing slashes
ifneq ($(filter %/, $(KBUILD_EXTMOD)),)
KBUILD_EXTMOD := $(shell dirname $(KBUILD_EXTMOD).)
endif

export KBUILD_EXTMOD

# backward compatibility
KBUILD_EXTRA_WARN ?= $(KBUILD_ENABLE_EXTRA_GCC_CHECKS)

ifeq ("$(origin W)", "command line")
  KBUILD_EXTRA_WARN := $(W)
endif

export KBUILD_EXTRA_WARN

# Kbuild will save output files in the current working directory.
# This does not need to match to the root of the kernel source tree.
#
# For example, you can do this:
#
#  cd /dir/to/store/output/files; make -f /dir/to/kernel/source/Makefile
#
# If you want to save output files in a different location, there are
# two syntaxes to specify it.
#
# 1) O=
# Use "make O=dir/to/store/output/files/"
#
# 2) Set KBUILD_OUTPUT
# Set the environment variable KBUILD_OUTPUT to point to the output directory.
# export KBUILD_OUTPUT=dir/to/store/output/files/; make
#
# The O= assignment takes precedence over the KBUILD_OUTPUT environment
# variable.

ifeq ("$(origin O)", "command line")
  KBUILD_OUTPUT := $(O)
endif

ifdef KBUILD_EXTMOD
    ifdef KBUILD_OUTPUT
        objtree := $(realpath $(KBUILD_OUTPUT))
        $(if $(objtree),,$(error specified kernel directory "$(KBUILD_OUTPUT)" does not exist))
    else
        objtree := $(abs_srctree)
    endif
    # If Make is invoked from the kernel directory (either kernel
    # source directory or kernel build directory), external modules
    # are built in $(KBUILD_EXTMOD) for backward compatibility,
    # otherwise, built in the current directory.
    output := $(or $(KBUILD_EXTMOD_OUTPUT),$(if $(filter $(CURDIR),$(objtree) $(abs_srctree)),$(KBUILD_EXTMOD)))
    # KBUILD_EXTMOD might be a relative path. Remember its absolute path before
    # Make changes the working directory.
    srcroot := $(realpath $(KBUILD_EXTMOD))
    $(if $(srcroot),,$(error specified external module directory "$(KBUILD_EXTMOD)" does not exist))
else
    objtree := .
    output := $(KBUILD_OUTPUT)
endif

export objtree srcroot
有一些疑问啊
abs_srctree其他我都理解 我就是不知道MAKEFILE_LIST是怎么发挥作用的。因为按照我的理解我只要能得到当前这个顶层makefile的全路径 后面dir和realpath我都看得懂 但是MAKEFILE_LIST可是迄今为止解析的所有的makefile文件列表路径啊 lastword提取的是最后一个 那能是当前也就是顶层的makefile的路径吗?毕竟在后面确实有include命令包含其他的makefile啊 只是啊 后面所有的include指令 全部依赖srctree的值 也就是全部要依赖abs_srctree或srcroot的值
然后关于KBUILD_EXTMOD的值怎么计算的以及srcroot的值怎么得到的 还有一个比较关键的变量是objtree的值是怎么计算的能一一详细地向我解析吗?

## ChatGPT:
好的，别担心，我会逐一详细地解析你提出的问题，包括 `MAKEFILE_LIST` 的作用、`KBUILD_EXTMOD`、`srcroot` 和 `objtree` 的计算。

**1. `MAKEFILE_LIST` 和 `abs_srctree`**

你对 `MAKEFILE_LIST` 的理解是正确的，它确实包含了到目前为止解析的所有 Makefile 文件的路径列表。你的疑问在于，为什么 `$(lastword $(MAKEFILE_LIST))` 提取的最后一个元素会是当前顶层 Makefile 的路径，而不是后面 `include` 包含的其他 Makefile 的路径。

这里的关键在于 Makefile 的解析顺序和 `include` 指令的处理方式：

*   **Makefile 的解析顺序**: `make` 命令首先读取顶层 Makefile（通常是名为 `Makefile` 的文件）。
*   **`include` 指令**: 当 `make` 遇到 `include` 指令时，它会暂停当前 Makefile 的解析，转而去读取并解析被包含的 Makefile 文件。被包含的 Makefile 文件会被视为当前 Makefile 的一部分，就像它们直接写在 `include` 指令的位置一样。
*   **`MAKEFILE_LIST` 的更新**: 每次 `make` 开始解析一个新的 Makefile 文件（无论是顶层 Makefile 还是被 `include` 包含的 Makefile），它都会将这个文件的路径添加到 `MAKEFILE_LIST` 变量的末尾。

因此，即使后面有 `include` 指令，`$(lastword $(MAKEFILE_LIST))` 在顶层 Makefile 中被计算时，`MAKEFILE_LIST` 中最后一个元素仍然是顶层 Makefile 的路径，因为 `include` 指令还没有被执行。

**`abs_srctree` 的计算**

现在我们来看 `abs_srctree` 的计算：

```makefile
this-makefile := $(lastword $(MAKEFILE_LIST))
abs_srctree := $(realpath $(dir $(this-makefile)))
```

1.  **`this-makefile := $(lastword $(MAKEFILE_LIST))`**:
    *   `$(lastword ...)` 函数提取列表中的最后一个单词（即最后一个 Makefile 文件的路径）。
    *   `this-makefile` 变量被赋值为当前 Makefile 的路径。

2.  **`abs_srctree := $(realpath $(dir $(this-makefile)))`**:
    *   `$(dir ...)` 函数提取路径中的目录部分，去掉文件名。例如，如果 `$(this-makefile)` 是 `/path/to/Makefile`，则 `$(dir $(this-makefile))` 的结果是 `/path/to/`。
    *   `$(realpath ...)` 函数将相对路径转换为绝对路径，并解析符号链接。
    *   `abs_srctree` 变量被赋值为当前 Makefile 所在目录的绝对路径。

所以，`abs_srctree` 最终存储的是顶层 Makefile 所在目录的绝对路径。

**2. `KBUILD_EXTMOD` 的计算**

`KBUILD_EXTMOD` 的计算涉及到 Makefile 的条件判断和变量的来源：

```makefile
ifeq ("$(origin M)", "command line")
  KBUILD_EXTMOD := $(M)
endif

ifeq ("$(origin MO)", "command line")
  KBUILD_EXTMOD_OUTPUT := $(MO)
endif

$(if $(word 2, $(KBUILD_EXTMOD)), \
	$(error building multiple external modules is not supported))

$(foreach x, % :, $(if $(findstring $x, $(KBUILD_EXTMOD)), \
	$(error module directory path cannot contain '$x')))

# Remove trailing slashes
ifneq ($(filter %/, $(KBUILD_EXTMOD)),)
KBUILD_EXTMOD := $(shell dirname $(KBUILD_EXTMOD).)
endif

export KBUILD_EXTMOD
```

1.  **`ifeq ("$(origin M)", "command line")`**:
    *   `$(origin ...)` 函数返回变量的来源。如果变量 `M` 是在命令行中定义的（例如 `make M=/path/to/module`），则 `$(origin M)` 的值为 "command line"。
    *   `ifeq (a, b)` 是 Makefile 的条件判断，如果 `a` 和 `b` 相等，则条件为真。
    *   如果 `M` 是在命令行中定义的，则 `KBUILD_EXTMOD` 被赋值为 `M` 的值。

2.  **`ifeq ("$(origin MO)", "command line")`**:
      如果MO变量是在命令行定义的, 则KBUILD_EXTMOD_OUTPUT被赋值为MO的值

3.  **`$(if $(word 2, $(KBUILD_EXTMOD)), ...)`**:检查是否提供了多个模块目录，如果提供了就报错

4.  **`$(foreach x, % :, ...)`**: 检查模块目录中是否包含非法字符

5.  **`ifneq ($(filter %/, $(KBUILD_EXTMOD)),)`**:
    *   如果 `KBUILD_EXTMOD` 的值以 `/` 结尾，则删除末尾的 `/`。

6.  **`export KBUILD_EXTMOD`**: 将 `KBUILD_EXTMOD` 导出为环境变量，以便子 Makefile 可以使用它。

总之，`KBUILD_EXTMOD` 的值通常来自于命令行参数 `M`，或者在没有提供 `M` 的情况下，可能为空。

**3. `srcroot` 的计算**

```makefile
ifdef KBUILD_EXTMOD
    # ...
    srcroot := $(realpath $(KBUILD_EXTMOD))
    $(if $(srcroot),,$(error specified external module directory "$(KBUILD_EXTMOD)" does not exist))
    # ...
else
    # ...
    objtree := .
    output := $(KBUILD_OUTPUT)
endif
```

1.  **`ifdef KBUILD_EXTMOD`**:
    *   `ifdef` 指令检查变量是否已定义（并且值不为空）。
    *   如果 `KBUILD_EXTMOD` 已定义（通常表示正在构建外部模块），则执行 `ifdef` 块内的代码。

2.  **`srcroot := $(realpath $(KBUILD_EXTMOD))`**:
    *   如果 `KBUILD_EXTMOD` 已定义，`srcroot` 被赋值为 `KBUILD_EXTMOD` 的绝对路径。

3.  **`$(if $(srcroot),,$(error ...))`**: 检查路径是否存在

如果没有定义 `KBUILD_EXTMOD`，`srcroot` 不会被赋值（但在后续的代码中，`srcroot` 和 `objtree` 都会被导出）。

**4. `objtree` 的计算**

```makefile
ifdef KBUILD_EXTMOD
    ifdef KBUILD_OUTPUT
        objtree := $(realpath $(KBUILD_OUTPUT))
        $(if $(objtree),,$(error specified kernel directory "$(KBUILD_OUTPUT)" does not exist))
    else
        objtree := $(abs_srctree)
    endif
    # ...
else
    objtree := .
    output := $(KBUILD_OUTPUT)
endif
```

1.  **`ifdef KBUILD_EXTMOD`**: 如果定义了,则开始判断是否定义了KBUILD_OUTPUT
2.  **`ifdef KBUILD_OUTPUT`**:
    *   如果 `KBUILD_OUTPUT` 也已定义，则 `objtree` 被赋值为 `KBUILD_OUTPUT` 的绝对路径。
    *   如果 `KBUILD_OUTPUT` 未定义，则 `objtree` 被赋值为 `abs_srctree`（顶层 Makefile 所在目录的绝对路径）。
3.  **`else`**:
    *   如果 `KBUILD_EXTMOD` 未定义（通常表示正在构建内核本身），则 `objtree` 被赋值为 `.`（当前目录）。

**总结**

*   **`MAKEFILE_LIST`**: 包含到目前为止解析的所有 Makefile 文件的路径列表。
*   **`abs_srctree`**: 顶层 Makefile 所在目录的绝对路径。
*   **`KBUILD_EXTMOD`**: 通常来自于命令行参数 `M`，表示外部模块的源代码目录。
*   **`srcroot`**: 如果构建外部模块，则为 `KBUILD_EXTMOD` 的绝对路径；否则不会被赋值（但在后续会被导出）。
*   **`objtree`**: 如果构建外部模块且定义了 `KBUILD_OUTPUT`，则为 `KBUILD_OUTPUT` 的绝对路径；如果构建外部模块但未定义 `KBUILD_OUTPUT`，则为 `abs_srctree`；如果构建内核本身，则为 `.`（当前目录）。

希望这个详细的解析能够解答你的疑惑！如果你还有其他问题，请随时提出。


> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
那在我们没指定KBUILD_EXTMOD 也即我们运行顶层makefile脚本的时候 没有指定选项M(也就是编译内核本身)
那么objtree是.也就是当前目录 这个当前目录是什么呢? 是不是还是顶层makefile所在目录呢?


## ChatGPT:
是的，你的理解完全正确。

当没有指定 `KBUILD_EXTMOD`（即没有通过 `make M=...` 指定外部模块路径），也就是编译内核本身时：

```makefile
else
    objtree := .
    output := $(KBUILD_OUTPUT)
endif
```

- `objtree` 被设置为 `.`，表示当前目录。
- 这里的“当前目录”指的是**运行 `make` 命令时所在的目录**。

通常情况下，我们会在内核源代码树的根目录下运行 `make` 命令，也就是顶层 Makefile 所在的目录。所以，在这种情况下，`objtree` 的值 `.` 就代表了顶层 Makefile 所在的目录。

**更精确的解释**

更准确地说，`objtree` 代表的是 **make 进程的当前工作目录**。

- 默认情况下，make 进程的当前工作目录就是你运行 `make` 命令时所在的目录。
- 但是，你可以通过 `make -C <directory>` 选项来改变 make 进程的工作目录。例如，`make -C /path/to/kernel/source` 会让 make 进程在 `/path/to/kernel/source` 目录下执行。

**总结**

- 在编译内核本身（没有指定 `KBUILD_EXTMOD`）时，`objtree` 的值是 `.`，表示 make 进程的当前工作目录。
- 通常情况下，我们会从顶层 Makefile 所在的目录运行 `make` 命令，所以 `objtree` 通常就是顶层 Makefile 所在的目录。
- 但可以通过 `make -C` 选项来改变 make 进程的工作目录。

所以，虽然通常情况下 `objtree` 就是顶层 Makefile 所在的目录，但更准确的说法是 make 进程的当前工作目录。


> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
那我们回到我们的起始的问题上 也就是顶层的makefile是如何为编译器添加全局的路径 使得<linux/xxx.h> 生效的
关键的代码应该就是LINUXINCLUDE  那这里我们应该就要根据之前的信息分类讨论了对吧 请帮助我详细分析一下。

## ChatGPT:
好的，我们回到最初的问题，分析顶层 Makefile 如何通过 `LINUXINCLUDE` 变量为编译器添加全局包含路径，使得 `#include <linux/xxx.h>` 能够生效。

正如你所说，这里的关键代码是 `LINUXINCLUDE` 变量的定义。我们需要根据之前讨论过的变量（`srcroot`、`objtree`、`KBUILD_EXTMOD`、`KBUILD_OUTPUT`）的不同情况来分类讨论。

```makefile
LINUXINCLUDE    := \
        -I$(srctree)/arch/$(SRCARCH)/include \
        -I$(objtree)/arch/$(SRCARCH)/include/generated \
        -I$(srctree)/include \
        -I$(objtree)/include \
        $(USERINCLUDE)
```

**分类讨论**

我们主要根据是否构建外部模块（`KBUILD_EXTMOD` 是否定义）以及是否指定了输出目录（`KBUILD_OUTPUT` 或 `O` 是否定义）来分类讨论。

**情况 1：构建内核本身，未指定输出目录**

*   **`KBUILD_EXTMOD`**: 未定义
*   **`KBUILD_OUTPUT` 或 `O`**: 未定义
*   **`srcroot`**: 未定义（但在稍后会被导出为顶层 Makefile 所在目录）
*   **`objtree`**: `.` (当前目录，即顶层 Makefile 所在目录)
*    **`srctree`**: 顶层 Makefile 所在目录

在这种情况下，`LINUXINCLUDE` 变为：

```makefile
LINUXINCLUDE    := \
        -I<顶层Makefile目录>/arch/$(SRCARCH)/include \
        -I<顶层Makefile目录>/arch/$(SRCARCH)/include/generated \
        -I<顶层Makefile目录>/include \
        -I<顶层Makefile目录>/include \
        $(USERINCLUDE)
```

**情况 2：构建内核本身，指定了输出目录**

*   **`KBUILD_EXTMOD`**: 未定义
*   **`KBUILD_OUTPUT` 或 `O`**: 已定义
*   **`srcroot`**: 未定义
*   **`objtree`**: `$(KBUILD_OUTPUT)` 的绝对路径
*   **`srctree`**: 顶层 Makefile 所在目录

在这种情况下，`LINUXINCLUDE` 变为：

```makefile
LINUXINCLUDE    := \
        -I<顶层Makefile目录>/arch/$(SRCARCH)/include \
        -I<输出目录>/arch/$(SRCARCH)/include/generated \
        -I<顶层Makefile目录>/include \
        -I<输出目录>/include \
        $(USERINCLUDE)
```

**情况 3：构建外部模块，未指定输出目录**

*   **`KBUILD_EXTMOD`**: 已定义
*   **`KBUILD_OUTPUT` 或 `O`**: 未定义
*   **`srcroot`**: `$(KBUILD_EXTMOD)` 的绝对路径
*   **`objtree`**: `$(KBUILD_EXTMOD)` 的绝对路径 (这里内核代码为了向后兼容有些复杂, 但总之是外部模块的绝对路径)
*   **`srctree`**: `$(KBUILD_EXTMOD)` 的绝对路径

在这种情况下，`LINUXINCLUDE` 变为：

```makefile
LINUXINCLUDE    := \
        -I<外部模块目录>/arch/$(SRCARCH)/include \
        -I<外部模块目录>/arch/$(SRCARCH)/include/generated \
        -I<外部模块目录>/include \
        -I<外部模块目录>/include \
        $(USERINCLUDE)
```

**情况 4：构建外部模块，指定了输出目录**

*   **`KBUILD_EXTMOD`**: 已定义
*   **`KBUILD_OUTPUT` 或 `O`**: 已定义
*   **`srcroot`**: `$(KBUILD_EXTMOD)` 的绝对路径
*   **`objtree`**: `$(KBUILD_OUTPUT)` 的绝对路径
*    **`srctree`**: `$(KBUILD_EXTMOD)` 的绝对路径

在这种情况下，`LINUXINCLUDE` 变为：

```makefile
LINUXINCLUDE    := \
        -I<外部模块目录>/arch/$(SRCARCH)/include \
        -I<输出目录>/arch/$(SRCARCH)/include/generated \
        -I<外部模块目录>/include \
        -I<输出目录>/include \
        $(USERINCLUDE)
```

**总结**

无论哪种情况，`LINUXINCLUDE` 变量都包含了几个重要的目录：

1.  **`arch/$(SRCARCH)/include`**: 特定体系结构的头文件目录。
2.  **`arch/$(SRCARCH)/include/generated`**: 自动生成的头文件目录。
3.  **`include`**: 内核的公共头文件目录。

这些目录会被添加到编译器的包含路径中，使得 `#include <linux/xxx.h>` 这样的语句能够找到对应的头文件。

需要注意的是:

*   无论是 `srctree` 还是 `objtree`，它们都有 `include` 子目录，其中包含了内核的公共头文件。
*   `arch/$(SRCARCH)/` 目录是特定于体系结构的，`$(SRCARCH)` 变量会根据目标体系结构（如 x86、arm）展开。

通过 `LINUXINCLUDE` 变量，Makefile 为编译器设置了包含路径，使得内核代码中的 `#include` 指令能够正确解析。


> provided by [EasyChat](https://site.eqing.tech/)
