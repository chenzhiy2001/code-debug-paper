# 论文修改意见

## 11月21日 - wjb

1. ✅ 概述：动机并不是开发者希望用“GDB”等调试器来同时调试内核和用户态，而是需要进一步分析原生的问题，比如：在调试基于系统调用等操作系统强相关的程序时，已有的调试器无法对操作系统内部的函数调用进行进一步跟踪。主要面临的问题在于xxxx。
2. “操作系统在特权级切换时会进行页表的切换，进而导致断点调试信息丢失，从而导致调试失败”，需要先解释什么叫做特权级，什么叫做特权级切换？页表的切换是否核心的原因？改成符号表等其他调试信息切换是否更合适？
3. 断点组是你提出来的方法，而不是“利用”。每次在出现新名词的时候前面都要先介绍。
4. ✅ 第2节标题是动机，但你实际写的内容是问题的展开描述
5. “假设一位开发者正在用 GDB 调试一个操作系统”，需要描述清楚到底是什么样的场景，比如调试操作系统内核模块时不存在你这里描述的这些问题
6. 2节中最后写的一大段解决方法要放到第3节中写，这里只要简单一两句话提一下。具体的写法建议去参考已有文章的写法和结构
7. ✅ 第3节“方法”，这个标题的写法最好去参考一下别人的文章中是怎么取的标题。
8. ✅ “将介绍实现该模块的方法”方法是方法，实现是实现，这是两码事，不要混为一谈
9. 在写方法之前需要有相关工作
10. 第3节需要以总分总的形式来组织，分小节描述核心问题，并且需要补图
11. 第4节，实现部分应该要从总体角度先分析框架，再写具体核心模块的实现，并且要补充例子

## 12月4日 - czy

上下文切换来的 灵感 （先介绍一下上下文切换，其中就涉及到页表切换和特权级切换，刚好能引出下文）
虽然断点不行，但是在一定条件下可以单步过去。
断点 作用域
本质上还是符号表的问题

第二章补充：不可以同时被调试器加载的（我们将在第二章中证明这一点）。

每个符号表有作用域，一个执行环境对应一个符号表，之前是因为可以共存符号表，没想过来。

先说不能跨特权级设置。后面具体分析。

解决方案可以先说怎么找到并替换合适用户符号表的问题，毕竟这个问题简单直接一点。然后引出解决方案钩子断点，这个就可以放在前面说了，减小后面累赘的内容。钩子断点要解决的是符号表的问题，之前没注意过

结构清晰之后就可以多加小标题了，后面的一些修改意见就可以自然地满足