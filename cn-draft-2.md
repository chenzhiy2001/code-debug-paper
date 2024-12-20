# 支持Rust语言的源代码级操作系统调试工具

- 文章解决的核心问题：操作系统代码调试时，特权级切换导致符号表等调试信息丢失进而断点调试失败。（需要确认：内核架构是否有影响，比如宏内核、微内核等）

- 核心方法：
  1. 利用断点组缓存不同特权级的断点等调试信息
  2. 特权级切换检测，以及切换时的调试信息切换

- 实现部分：
  - 基于vscode插件统一真实硬件和qemu模拟器的调试

## 摘要

## 1 Introduction 概述

在远程调试（cross-debugging）操作系统时，现有的调试器只能单独调试操作系统内核或运行于该操作系统中的单个用户程序，无法同时调试内核和多个用户程序。具体来说（to be specifically），用户不能在被调试 OS 暂停在内核时设定某个用户程序的断点，也不可以在被调试 OS 暂停在一个用户程序时设置内核或者另一个用户程序的断点。

造成这种不便的原因是，操作系统为了防止用户程序对操作系统造成损害以及提高硬件资源的利用率，在内核和每个应用程序之间都做了隔离。隔离的措施主要有两个：1）内核代码和用户程序代码拥有不同的执行权限，换句话说，内核运行在内核特权级，用户程序运行在用户特权级；2）内核和每一个用户态程序都拥有独立的上下文（context），特别是独立的地址空间，这意味着内核和每一个应用程序的符号表都是互相独立的。

当互相隔离的内核和用户程序需要交互时，它们的执行流会通过系统调用和陷入返回（trap return）的方式切换到目标内核/用户程序。在这种隔离环境切换的过程中，除了处理器的特权级会切换到目标内核/用户程序的特权级以外，包括页表在内的上下文也会进行切换。但是由于调试器不能识别到这种隔离环境的切换，所以它不能在页表切换之后将调试器加载的符号表替换为新隔离环境的符号表，因此衍生出了无法跨隔离环境设置断点的问题。因此，如果调试器同时加载内核和多个用户程序的符号表并设置非当前隔离环境的断点，那么这些断点很有可能设置失败。我们将在第二章中证明这一点。

当用户程序需要执行一些特权操作或与操作系统交互时，它需要通过系统调用的方式从用户态切换到内核态。但另一方面，特权级的切换会清除调试状态信息，使得传统程序调试方法失效。

上述这些调试器对操作系统源代码的调试支持不够完善，其中一个主要问题在于操作系统不同运行状态下程序符号表的切换导致断点等调试信息丢失，从而使得调试器失效。

首先，操作系统中包括用户态代码和内核态代码，对应不同的特权级，不同的特权级又对应不同的符号表。在操作系统运行的过程中，会频繁地进行特权级的切换，导致调试信息的丢失，而目前现有的调试器都无法进行跨特权级的断点调试。具体来说，特权级的切换主要涉及符号表的切换，符号表包含了编译后的代码中各种变量、函数、数据结构等的名称和地址信息，这些是调试代码所必需的内容，同时符号表也是编译后的代码与源代码之间的桥梁，使调试器能够将二进制代码中的地址映射回源代码的符号名，所以无论调试任何进程，调试器都需要先加载进程的符号表。而在操作系统中内核态程序和用户态程序的符号表是分开的，如果程序运行中进行了用户态和内核态的转换，符号表也要随之切换，符号表切换以后，用户设置的程序断点也会随之消失，比如在内核态设置用户态的断点以后，再进入用户态，用户态的断点将不会被触发。

其次，用户态中的多个用户进程分别对应不同的符号表，在CPU对用户进程进行调度切换运行状态时同样存在上述的调试信息失效的问题。如何解决操作系统中多符号表切换导致的调试信息失效是本专利解决的关键问题。

对于被隔离的内核和某个用户态程序来说，它们有不同特权级，交互要特权级切换，切换的时候上下文就要切换。

思路：隔离，隔离导致上下文不同，切换隔离的时候上下文也切换，而调试器既不识别隔离也不识别切换，因此导致问题。

要跨过隔离，

然而，调试器没有这种隔离。这就带来了大量困扰，后面详述。

为了解决，引入隔离，断点同样做隔离，有不同的上下文。有上下文切换。

操作系统开发者希望用 GDB 调试器或者它的继承调试环境来同时调试内核和多个用户态程序，而且断点是调试过程中一定会用到的功能。

然而，操作系统在特权级切换时会进行页表的切换，进而导致断点调试信息丢失，从而导致调试失败。

为了解决上述问题，我们利用断点组缓存不同特权级的断点等调试信息，同时通过边界断点进行特权级切换检测，从而能够在特权级切换时进行调试信息的切换，进而实现跨特权级，多进程的操作系统调试。

我们基于一个 VSCode 插件，在真实硬件和 Qemu 模拟器上实现了这种操作系统调试。

## 相关工作

### 调试器

### 特权级切换

### 符号表

## 2 问题的展开描述

先说符号表：没符号表根本就无法设置源代码断点。

就算加载符号表，由于页表的问题，断点也无法跨特权级设置。

我们不妨假设一位开发者正在用 GDB 调试一个操作系统。该操作系统已经暂停在内核代码，且暂未运行过任何用户态进程。此时他想调试该操作系统将要进入到的第一个进程，即零号进程的代码。于是他加载了零号进程的符号表，并在 GDB 终端里输入了一行命令，请求设置一个位于零号进程的代码的断点。我们暂时称这个进程为 $ P_1 $，这个断点为 $ B_1 $ [^1]。

GDB在执行这个命令时会依次做这几件事：

1. 从已经加载的符号表中推断出该断点的虚拟内存地址。我们暂时称这个这个内存地址为 $ A_1 $。
2. 将虚拟内存地址 $ A_1 $ 处的指令修改为一个特殊的中断指令（这个指令在 x86 处理器上是 `INT3`，在 riscv 处理器上是`ebreak`）。

在执行步骤2时，如果内核和用户态程序共用一个页表，那么不论操作系统运行在内核还是该用户态程序，虚拟地址 $ A_1 $ 都指向断点 $ B_1 $ 对应的二进制指令，因此断点 $ B_1 $ 可以成功地设置并触发。但是，如果内核和用户态程序使用不同的页表，那么会出现两种不寻常的情况：

1. 虚拟内存地址 $ A_1 $ 只被包含在零号进程的页表里，不被包括在内核的页表里。在这种情况下，在内核态下，虚拟内存地址 $ A_1 $ 是无法访问到的，因此 GDB 会认为这个断点目前无法设置，并将这个断点设置为 `PENDING` 状态，每一次被调试操作系统因单步或断点而停下来时，就重新尝试设置一次这个断点。

2. 虚拟内存地址 $ A_1 $ 不仅被包含在零号进程的页表里，也被包括在内核的页表里，但是在内核的页表里，虚拟地址 $ A_1 $ 并没有像零号进程的页表一样指向断点 $ B_1 $ 对应的二进制指令，而是指向其他无关内容。由于内核态下 $ A_1 $ 并不是断点 $ B_1 $ 对应的二进制指令的位置，GDB 会将一个无关指令（甚至可能是一些无关数据）修改为中断指令，在错误的物理地址处设置断点。

这两种不寻常的情况意味着，由于 GDB 不支持跨页表的断点设置，但是操作系统进行特权级切换后会切换页表，GDB 不能进行跨特权级的断点设置。也就是说，如果内核和用户态程序使用不同的页表，那么 GDB 就不能同时设置内核和用户态程序的断点。

如果用户想利用断点同时调试被调试操作系统中的**多个**用户态进程，那么不论内核和用户态程序是否共用一个页表，GDB 都不能正确地同时设置不同用户态进程的断点。例如，假设用户在断点 $ B_1 $ 触发后命令 GDB 设置另一个进程 $ P_4 $ 的断点 $ B_4 $，GDB 会根据已有符号表解析出断点 $ B_4 $ 对应的内存地址 $ A_4 $ 并修改地址 $ A_4 $ 处的指令。然而，由于不同用户态程序的地址空间是重叠的（overlapping），在用户设置断点 $ B_4 $ 的这个时机，断点 $ B_4 $ 对应的地址 $ A_4 $ 指向的是当前运行的进程 $ P_1 $ 的某个指令，而非进程 $ P_4 $ 的指令。

综上所述，在调试操作系统时，由于 GDB 不能正确地设置非当前页表的断点，而操作系统却有频繁的页表切换，仅靠 GDB 进行操作系统的断点调试是不够的。

针对这个问题，我们提出了一个解决办法：由一个外部模块（我们称之为断点组管理模块）保存所有的断点信息，该模块会将所有的断点按照它们所属的页表进行分组，并且保证 GDB 只设置当前页表下的那一组断点（我们称这一组断点为当前断点组）。非当前页表的断点均缓存在断点组管理模块中，不为 GDB 所知。例如，如果用户设置断点 $ B_4 $ 的命令在 GDB 执行它之前经过断点组管理模块的过滤，那么该模块就会检测到断点 $ B_4 $ 不属于当前运行的进程 $ P_1 $ 的页表，进而不将设置断点 $ B_4 $ 的命令传递给 GDB，而是将它缓存到进程 $ P_4 $ 的页表对应的断点组中。等到断点组管理模块检测到 OS 已经切换到进程 $ P_4 $ ，断点组管理模块会命令 GDB 删掉旧的断点组的断点，并设置进程 $ P_4 $ 对应的断点组的断点，而这个断点组中就包含了断点 $ B_4 $。

既然断点 $ B_4 $ 属于进程 $ P_4 $，那么断点 $ B_4 $ 只有可能在 OS 运行进程 $ P_4 $ 时被触发，所以在 OS 切换到进程 $ P_4 $ 之后立即设置包含断点 $ B_4 $ 的断点组，且在 OS 因为系统调用或进程结束而离开进程 $ P_4 $ 之后令 GDB 把包含断点 $ B_4 $ 的断点组的断点删掉，使这些断点组恢复到缓存状态（即断点组管理模块中有这些断点组的内容，但是 GDB 中没有设置这些断点组里的断点），可以保证断点 $ B_4 $ 正确地被设置和完全地被触发。

[^1]: 本文不讨论硬件断点（hardware breakpoint），只讨论软件断点（software breakpoint）的情况。

## 3 设计

在上一章中，我们提出利用断点组管理模块来解决跨特权级，多进程断点调试中出现的问题。在本章中，我们将介绍该模块的详细设计。

### 3.1 符号表

### 3.2 单进程

### 3.3 多进程

上一章我们提到我们要用一个断点组管理模块来进行断点的管理，进而实现只在当前页表下设断点，这样就可以保证不出现上一章所述的双页表 OS 跨特权级设断点及单页表 OS 跨进程设断点时出现的问题。

首先要考虑的问题是断点组管理模块中的数据结构。我们用一个词典保存所有的断点组。词典中某个项（entry）的键是断点组的名字，值是一个数组，其中包含了属于该断点组的所有断点的信息。

现在首要的问题是，我们如何判断出什么样的断点是属于当前页表的。既然断点组管理模块可以拦截用户设置断点的命令，而用户设置断点的命令中，通常包含了断点的源代码文件名和行号（在此处我们暂不讨论直接设置内存地址断点的情况），那么断点组管理模块就应当从用户设置断点的命令中的信息来解析出这个断点所对应的页表。（让用户自己去设置的原因时，这玩意没有固定的规则），因此我们希望用户在配置文件中提交一个规则，这个规则指定了断点的源代码文件名到断点组名的映射。例如，如果所有在“initproc”文件夹下的代码文件都属于initproc进程的页表，那么用户可以在配置文件中将“initproc”文件夹下的代码文件都映射到一个叫做“initproc_breakpoints”的断点组。有了这样的映射之后，每一个断点都可以被归类到对应的断点组里。

由于本模块的核心功能是保证只设置当前页表下的断点，而我们已经有了一个将断点设置命令归类到不同页表的机制，那下一步要做的是确定哪一个断点组是 OS 当前的断点组。

由于在一些处理器，比如risc-v中，并没有一个寄存器可以直接指定当前特权级，所以在一些情况下，我们无法直接判断出当前所在的特权级及页表信息。我们采取的办法是，在每一个断点组中添加一个我们称为出口断点的断点。这个断点的作用是，一旦这个断点触发了，就表示操作系统即将离开当前的断点组。例如，内核断点组的出口断点设置在内核的上下文切换代码，用户态程序所属的断点组的出口断点设置在syscall指令处。在操作系统调试开始时，断点组管理模块的当前断点组默认为内核断点组。一旦边界断点触发，断点组管理模块就会将当前断点组切换到下一个断点组。获知哪个断点组是下一个断点组的方式是，在断点组管理模块内部保存有两个属性，第一个为“下一个断点组”，第二个为“下下一个断点组”。这两个属性的初始值是由用户在配置文件中指定的。一般情况下，用户应当配置“下一个断点组”为零号进程所属的断点组，“下下一个断点组”为内核断点组（因为操作系统进入零号进程后会返回内核）。例如，操作系统运行在内核时，“下一个断点组”是零号进程断点组，当操作系统从内核切换到零号进程后，断点组调试模块会将“下一个断点组”和“下下一个断点组”的值互换，使得当操作系统从零号进程返回到内核前，“下一个断点组”为内核断点组。当操作系统从零号进程返回内核后，“下一个断点组”和“下下一个断点组”再次进行对调，使得“下一个断点组”变为零号进程断点组，“下下一个断点组“变成内核断点组。此时“下一个断点组”的值是错误的。我们利用我们称之为“钩子断点”的特殊断点在内核代码执行期间获取到正确的下一个断点组名并赋值给“下一个断点组”变量。“钩子断点”被触发时，GDB 调试器会自动执行一些用户自定义的行为。这些用户自定义的行为会捕捉到下一个断点组的正确的值。例如，用户可以把钩子断点设置在内核的调度代码，并在钩子断点的行为函数中，将该断点触发后的行为设置为捕获某个变量代表的字符串，并将该字符串用用户设定的规则进行拼接，进而得到正确的“下一个断点组”变量的值。因此，操作系统在从零号进程返回内核之后会触发钩子断点，钩子断点捕获到下一进程名后，断点组管理模块中的“下一个断点组”变量会从“零号进程断点组”切换为下一个要运行的进程的断点组名。

当出口断点被触发后，我们除了要将“当前断点组”变量更新，还需要将断点组本身进行切换。由于 GDB 不能在被调试对象还在运行时设置断点，我们需要在每次页表被切换后停下来，接着切换断点组和符号表。一个很容易想到的办法是，在页表切换后的指令处设置断点（我们暂时称这个断点为 $ B_2 $ ），当 $ B_2 $ 被触发时就意味着页表已经发生了切换，在这个时机切换断点组即可。但是这个办法在逻辑上是不可行的，因为断点 $ B_2 $ 会遇到和断点 $ B_1 $ 同样的问题，无法被设置或触发。在页表切换指令或其之前的几个指令处设断点（我们暂时称这个断点为 $ B_3 $ ）同样是不可行的：虽然由于断点 $ B_3 $ 本身和设置它时操作系统所在的页表是一致的，因此可以被正确地设置和触发，但是它被触发时尚未进行页表的切换，此时若切换到下一断点组，那么下一断点组的所有的断点都会遇到和断点 $ B_1 $ 同样的问题，无法被设置或触发。

由此可见，只是设置一个新的断点是不足以帮助完成断点组的切换的。但是我们注意到，断点 $ B_3 $ 和断点 $ B_2 $ 分别可以完成切换断点组的一半工作：

1. 如果操作系统暂停在断点 $ B_3 $ 所在的指令处（如前文所述，断点 $ B_3 $ 本身和设置它时操作系统所在的页表是一致的，所以让 GDB 设置断点 $ B_3 $ 就可以促成这个情况的发生），那么 GDB 可以成功地卸载旧断点组的断点，因为此时旧断点组的断点所属的页表和卸载旧断点组的断点时操作系统所在的页表是一致的；

2. 如果操作系统能够暂停在断点 $ B_2 $ 所在的指令处（尽管我们无法通过提前设置断点 $ B_2 $ 来促成这个情况的发生，但是可以在断点 $ B_3 $ 触发后进行数次单步来促成这个情况的发生），那么 GDB 可以成功地设置新断点组的断点，因为此时新断点组的断点所属的页表和加载新断点组的断点时操作系统所在的页表是一致的；

因此，GDB 只需要设置断点 $ B_3 $，断点 $ B_3 $ 被触发后再利用单步操作到达断点 $ B_2 $ 所在的指令处，就可以利用这两次被调试操作系统的暂停共同完成断点组的切换，避开了无法提前设置断点 $ B_2 $ 的问题。当操作系统运行在某一页表下时，先设置这个页表下的断点 $ B_3 $（我们称这个断点为“出口断点”，因为它的触发意味着操作系统即将离开当前的页表，因此需要进行一部分断点组切换的工作）。在断点 $ B_3 $ 被触发后，GDB 卸载旧断点组的断点，然后让 GDB 不断地进行单步操作，直到它暂停在断点 $ B_2 $ 所在的指令处。接着，GDB 再设置新断点组的断点。

需要注意的是，由于断点组管理模块中保存的断点都是源代码断点，而设置一个源代码断点之前，它对应的符号表必须已经加载，因此，在设置新断点组的断点之前，断点组管理模块需要查找到新断点组依赖的符号表，并且让 GDB 加载这些符号表。类似地，在卸载完旧断点组的断点之后，旧断点组的符号表需要卸载掉，以免和新断点组的符号表产生冲突。断点组和它对应的符号表文件的映射关系也需要用户在配置文件中提交。

## 4 Implementation 实现

设计上优先符号表，但是由于符号表的xx依赖断点，所以先实现断点。

### A. 修改 Native Debug 插件

接下来的内容：

- 修改已有的用于调试用户态程序的 Native Debug VSCode 插件，为其增加Qemu支持。一键启动 Qemu + GDB。
- 修改调试器插件已有的断点设置和断点触发处理函数，接入断点组管理模块
- 一个断点组管理模块要有哪些元素：断点组，断点映射到断点组，断点组映射到符号表，出口断点
- 出口断点是人工设置的（如果要自动设置的话，需要在断点组管理模块里支持基于地址的断点，并在可执行文件里找到ecall指令，我们没这么干）
- 断点 $ B_2 $ 对应的指令位置是自动检测的，检测方式是，每单步一次，就将当前的 PC 寄存器值和配置文件中提交的新断点组的 PC 寄存器地址范围做匹配，（因为页表切换后，PC寄存器值会有很大的变动，所以这么做可行），若匹配，则页表已经切换，断点组管理模块视为已经单步到断点 $ B_2 $ 对应的指令位置，开始加载新断点组。
- 出口断点 $ B_3 $ 不一定非得是页表切换指令的前一个指令，再前面一点也可以，只要保证单步的过程不被中断打断，出口断点触发之后一定会进行页表切换即可。好在页表切换之前，大部分操作系统都会关中断。关完中断之后的指令都可以当出口断点。因为出口断点是源代码断点，可能不能很精确地打到某一个指令（因为一行代码往往代表多个指令），所以我们设计的断点组切换方案对出口断点位置的要求比较宽松。
- 如果用户在调试过程中动态增加或删减了某个断点
- 限制：这种机制是同步的，所以异步的异常无法跟踪
- 每单步一次都判断pc范围，来确定有没有到达终点

### B. 状态机，多进程

接下来的内容：

- 多进程调试的流程是什么样的，调试器需要增加什么代码
- 由于添加多进程调试功能，代码维护起来比较繁琐，所以我们用一个状态机来整理代码，把以前的所有的跨特权级调试功能和多进程调试功能都囊括进去。这个状态机包括哪些 states, events, actions，分别起什么用途
- 在状态机之外，还有钩子断点（下一个用户态进程的问题），起什么作用

### C. 硬件调试

接下来的内容：

- 基本思路：用 OpenOCD 替换 Qemu 的 GDBStub
- RISC-V 的 stepie 的问题

## 5 Related Work 相关工作

## 6 Conclusion 结论
