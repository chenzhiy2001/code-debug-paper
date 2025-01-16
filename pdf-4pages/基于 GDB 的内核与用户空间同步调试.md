**Debugging Kernel and User Space Synchronously Based on GDB**  
Zhiyang Chen, Jingbang Wu*, Yaqi Yang, Yifan Zhang  
School of Computer and Artificial Intelligence  
Beijing Technology and Business University  
Beijing, China  
*Corresponding author: wujingbang@btbu.edu.cn  

Hongyuan Wang  
Faculty of Information Technology  
Beijing University of Technology  
Beijing, China  

---

## Abstract  
Debugging operating systems with multiple privilege levels is difficult due to symbol table mismatches from context switching. Conventional debuggers struggle to maintain breakpoints and symbol tables as the OS transitions between kernel and multiple user processes. To solve this, we propose a method based on breakpoint groups and automated switching. By grouping and caching breakpoints and symbol tables, then activating the relevant group at runtime, consistent debugging is preserved across varied execution contexts. Experiments on real RISC-V hardware and QEMU verify its effectiveness on OSes like xv6 (C) and Starry (Rust/C). Our solution offers a practical, portable approach for cross-privilege, multi-process debugging in both academic and real-world OS development.

**Keywords**—Operating System Debugging; GDB;

---

## I. INTRODUCTION  
When debugging an operating system (OS) from outside—via QEMU’s GDBStub [1] or similar interfaces—traditional debuggers can target only the kernel or a single user program at once. They cannot debug kernel and multiple user programs simultaneously. Specifically, users cannot set a user-level breakpoint if the OS is paused in kernel mode, or a kernel breakpoint if the OS is paused in a specific user process.

> This work is supported by the National Natural Science Foundation of China (NSFC) under Grant No. 62402019 and Beijing Natural Science Foundation under Grant No.4244072.

This limitation arises from kernel-process isolation aimed at preventing user programs from harming the OS and optimizing hardware resource usage. Two key isolation measures are:  
1) The kernel runs at kernel privilege level, while user processes run at user privilege level.  
2) The kernel and each user process have distinct contexts, notably separate address spaces.  

When switching among these isolated environments (via system calls, trap returns, etc.), the processor’s privilege level and page table context also switch. Most breakpoints are set at the source level, requiring a corresponding symbol table. Because each isolated environment has its own symbol table, any mismatch between loaded symbol table and active address space invalidates breakpoints.  

We address this by grouping symbol tables (and the breakpoints depending on them) by isolated environment. Each group, called a “breakpoint group” [2], is tied to a specific symbol table. At runtime, only the active group (kernel or a particular user process) is loaded. This prevents invalid breakpoints.  

We implement this via two key methods. First, we cache breakpoints and symbol tables into groups, guided by two mapping rules provided by the user:  
- **Rule 1**: Maps a breakpoint group name to the path of its symbol table file.  
- **Rule 2**: Maps source code filenames to breakpoint group names.  

When a user sets a breakpoint, the debugger derives the breakpoint group from Rule 2, caches the breakpoint there, and sets it immediately if that group is currently active.  

Second, we detect environment switches using two special breakpoint types:  
- **Hook breakpoints** execute user-defined commands after they trigger, enabling the debugger to capture OS context (e.g., the name of the next process).  
- **Exit breakpoints** indicate impending departure from the current environment (e.g., system call entry in user space).  

Before debugging starts, users specify filenames and line numbers for hook and exit breakpoints, plus how to retrieve context, in a configuration file. When a hook breakpoint triggers, the debugger gathers context (e.g., the next process). Upon reaching an exit breakpoint, the debugger switches breakpoint groups by unloading the old group and loading the new group after single-stepping past the page-table-switch instruction. We elaborate in Chapter III.  

We implemented a VSCode plugin for both real hardware and QEMU to realize this mechanism. Details are in [3].

---

## II. RELATED WORK  

### A. Breakpoints and Single-Stepping Mechanism  
Debuggers like GDB [4], LLDB, or WinDBG typically set breakpoints by replacing the target instruction with a special interrupt (e.g., `INT3` on x86, `EBREAK` on RISC-V). Executing this instruction triggers an exception, pausing execution and returning control to the debugger, which can inspect CPU state, memory, etc. If the target instruction is inaccessible or mapped differently (due to page tables), the breakpoint may fail.

Single-stepping uses a processor flag or software simulation to pause after each instruction, avoiding the need to replace instructions. It is often used when setting a breakpoint at a certain address is infeasible.

### B. Symbol Tables  
Source-level debugging depends on debugging information (e.g., DWARF in ELF files [5]), which map source-level identifiers (functions, variables, line numbers) to virtual addresses. When a user sets a breakpoint in source code, the debugger consults the symbol table to find the corresponding virtual address and patches in the special interrupt instruction.

---

## III. PROBLEM DESCRIPTION  
Here, we explain how page table switches when transitioning between the kernel and user-mode processes thwart cross-privilege, multi-process breakpoints, motivating our design of the Breakpoint Group Management Module in Chapter IV.

Suppose a developer debugs an OS running on real hardware or QEMU [6], connected via GDB (through OpenOCD [7] on hardware or QEMU’s GDBStub). The OS is paused in kernel code, and no user process has run yet. The developer wishes to debug process 0 (the first user process) by setting a breakpoint B1 in that process (call it P1) using the user program’s symbol table. GDB:  
1) Computes the virtual address A1 for B1 from P1’s symbol table.  
2) Patches the instruction at A1 with a special interrupt (e.g., `EBREAK` on RISC-V).  

If kernel and user processes share a page table, A1 is valid regardless of the privilege level. However, if they have separate page tables, two scenarios may occur:  
- A1 is valid only in P1’s page table, not the kernel’s. GDB marks B1 as **PENDING** and keeps trying to set it whenever execution stops.  
- A1 exists in both the kernel and P1 page tables, but in the kernel, A1 maps to an unrelated address. GDB ends up patching an irrelevant instruction or data region with the interrupt.  

Thus, if the OS changes page tables across privilege transitions, GDB alone cannot set breakpoints spanning kernel and user processes. This also applies to multiple user processes: GDB cannot set breakpoints in a non-current process’s address space if each process uses its own page table.

To solve this, we introduce an external module that groups and caches breakpoints for each environment and dynamically switches them (Fig. 1).

**Figure 1.** State Transitions and Actions for Managing Breakpoints Across Isolated Environments.

---

## IV. DESIGN  
We propose a Breakpoint Group Management Module to mitigate cross-environment breakpoint problems. It intercepts user “set breakpoint” requests before reaching GDB, grouping them by their respective page tables. Only the currently active group is handed to GDB; others remain cached.  

We first justify that caching breakpoints this way ensures correctness, then detail how we classify breakpoints, track the next environment, and execute group switches. Our design is flexible and not tied to any special debugging hardware, though it does require user configuration of mapping rules. Future enhancements could automate configuration.

### A. Correctness  
Consider the scenario from Chapter III where the user wants to set breakpoint B4 in process P4 while currently debugging process P1. Our module detects that B4 does not belong to P1’s page table, so it caches B4 in P4’s group rather than passing it to GDB. When the OS switches to P4, the module unloads P1’s breakpoints from GDB and loads P4’s, including B4. This ensures B4 is correctly set and triggered only when the OS is actually running P4.

### B. Caching Breakpoints into Breakpoint Groups  
When the user issues a breakpoint command referencing a filename and line, the module checks a “source filename → breakpoint group” mapping. The corresponding breakpoint group is determined, and if it matches the current environment, the command goes to GDB immediately. Otherwise, it is cached.  

Since we need to know which group is currently active, we insert **exit breakpoints** in each group to detect when the OS is about to exit that environment (e.g., kernel context switching code for the kernel group, system call entry for user-mode groups). Initially, we assume the kernel group is active. When its exit breakpoint triggers, the module switches to the next group.

### C. Determining the Next Breakpoint Group  
We maintain two properties: **next breakpoint group** and **next-next breakpoint group**, with initial values set by the user (e.g., the kernel’s “next” group might be process 0; the process 0 group’s “next” might be the kernel). After each environment switch, we swap these properties so the next exit will return to the correct group.  

However, if the OS can switch to any process, the previously set “next group” may be inaccurate. We use **hook breakpoints** in the kernel’s scheduling code, which, when triggered, run user-defined commands to capture the name/identifier of the next process. That identifier updates the “next breakpoint group” so the correct group loads on the next switch.

### D. Breakpoint Group Switching  
Upon hitting an **exit breakpoint**, we must not only update “current breakpoint group” but also properly unload the old group and load the new group. However, simply placing a new breakpoint at the post-switch instruction (B2) will fail for the same reason as B1. Likewise, placing a breakpoint before the switch (B3) triggers too early to load the new group.  

Instead, we use a two-step process:  
1) **Exit breakpoint (B3)** is set in the environment about to be left. When triggered, we unload the current group’s breakpoints and symbol tables.  
2) We then **single-step** repeatedly until reaching the instruction after the page table switch, conceptually the B2 location. Here, the environment is fully switched, so we load the new group’s symbol tables and breakpoints without errors.  

Symbol tables are swapped accordingly: before setting breakpoints for the new group, we load the new group’s symbol tables. After unloading the old group’s breakpoints, we unload its symbol tables. These mappings come from the user’s configuration file.

---

## V. IMPLEMENTATION  
We modified the Native Debug VSCode extension [9] to include our Breakpoint Group Management Module, hook breakpoint logic, exit breakpoints, and automated switching. Users can now simultaneously set kernel and multi-process breakpoints in a single debug session.

### A. Overall Framework and Breakpoint Group Management  
**Figure 2** shows the extension’s architecture: the VSCode Debug Adapter routes user commands to our module before forwarding them to GDB. The module class **BreakpointGroups** stores the dictionary of group names, breakpoints, and their associated symbol tables. It also tracks the current, next, and next-next groups. If the user adds or removes breakpoints during debugging, the module updates the relevant group and decides whether to immediately instruct GDB to reflect those changes.  

Transition events—especially exit breakpoints and hook breakpoints—update these group states. We implemented a finite state machine, **OSStateMachine**, to recognize transitions among kernel, user, and intermediate single-stepping states.

**Figure 2.** Overall Framework of our Implementation.

### B. State Machine for Debugging Context Transitions  
To manage breakpoint group switching, we define states like “kernel,” “user,” and transitional ones for single-stepping after exit breakpoints. When a stop occurs (due to a breakpoint or single-step), the state machine checks the event type and executes appropriate actions: unloading/loading groups, single-stepping, or reading the next process.  
  
Table I summarizes the main transitions:

**TABLE I. STATE TRANSITIONS AND ACTIONS FOR DEBUGGING CONTEXT SWITCHING**

| State (OSState)             | Possible Event (OSEvent)                          | Action                                                                                                                                                                    |
|-----------------------------|----------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **kernel**                  | **STOPPED**                                       | Update “next breakpoint group” if a hook breakpoint triggered; on exit breakpoint, trigger **AT_KERNEL_TO_USER_BORDER** event                                            |
| **AT_KERNEL_TO_USER_BORDER**| --                                                | Unload kernel group’s symbol tables/breakpoints; go to **kernel_single_step_to_user** state; start single-stepping                                                        |
| **kernel_single_step_to_user** | **STOPPED**                                    | If user mode is reached (check PC), trigger **AT_USER** event                                                                                                             |
| **AT_USER**                 | --                                                | Load user group’s symbol tables/breakpoints; swap next and next-next group names; transition to **user** state                                                           |
| **user**                    | **STOPPED**                                       | On exit breakpoint, trigger **AT_USER_TO_KERNEL_BORDER** event                                                                                                           |
| **AT_USER_TO_KERNEL_BORDER**| --                                                | Unload user group’s symbol tables/breakpoints; go to **user_single_step_to_kernel** state; start single-stepping                                                          |
| **user_single_step_to_kernel** | **STOPPED**                                    | If kernel mode is reached, trigger **AT_KERNEL** event                                                                                                                    |
| **AT_KERNEL**               | --                                                | Load kernel group’s symbol tables/breakpoints; swap next and next-next group names; transition to **kernel** state                                                        |

Hook breakpoints help determine the correct “next group” within kernel code before switching to user mode. Similar expansions can support additional privilege levels.

### C. Examples and Experiments  
We tested this cross-kernel/user, multi-process debugging method on various OSes (rCore-Tutorial [11], xv6 [12], Starry [13]) running on RISC-V hardware or QEMU. Table II shows the specific environments. Porting to other architectures or OSes generally requires adjusting exit/hook breakpoint locations, specifying how to identify the next process, and configuring “filename → group” and “group → symbol table” mappings in the configuration file.

**TABLE II. TESTED OPERATING SYSTEMS AND DEBUGGING ENVIRONMENTS**  

| Operating System           | Debug Interface                                      | CPU Architecture |
|----------------------------|------------------------------------------------------|------------------|
| rCore-Tutorial (Rust)      | QEMU GDBStub                                        | RISC-V           |
| xv6 (C)                    | QEMU GDBStub / VisionFive2 (JTAG & OpenOCD)         | RISC-V           |
| Starry (Rust/C Hybrid)     | QEMU GDBStub                                        | RISC-V           |

---

## VI. CONCLUSION  
We introduced a cross-privilege-level, multi-process OS debugging method. A Breakpoint Group Management Module caches breakpoints and symbol tables by environment, then switches them automatically when the OS transitions (kernel ↔ process). Exit breakpoints detect environment exits, while hook breakpoints in kernel code retrieve the next process identifier, supporting multi-user-process debugging. Tests show it effectively sets and triggers breakpoints across the kernel and multiple user programs on both QEMU and real RISC-V hardware, illustrating the approach’s portability.

---

## REFERENCES  
[1] H. Li, Y. Xu, F. Wu, and C. Yin, “Research of ‘Stub’ remote debugging technique,” in *Proc. 2009 4th Int. Conf. Computer Science & Education*, pp. 990–994, 2009.  
[2] Z. Chen, Y. Yu, Z. Li, and J. Wu, “An online debugging tool for Rust-based operating systems,” 2022.  
[3] Z. Chen, “Code Debug.” [Online]. Available: <https://github.com/chenzhiy2001/code-debug>  
[4] R. Stallman, R. Pesch, S. Shebs, and others, “Debugging with GDB,” *Free Software Foundation*, vol. 675, 1988.  
[5] DWARF Debugging Information Format Committee, “DWARF debugging information format version 4,” Tech. Rep., June 2010.  
[6] F. Bellard, “QEMU, a fast and portable dynamic translator,” in *Proc. USENIX Annu. Tech. Conf., FREENIX Track*, vol. 41, no. 46, pp. 10–5555, 2005.  
[7] H. Högl and D. Rath, “Open on-chip debugger—openocd,” Fakultat für Informatik, Tech. Rep., 2006.  
[8] Waterman, Y. Lee, R. Avizienis, D. A. Patterson, and K. Asanovic, “The RISC-V instruction set manual volume II: Privileged architecture version 1.7,” EECS Dept., Univ. of California, Berkeley, Tech. Rep. UCB/EECS-2015-49, 2015.  
[9] J. Jurzitza, “Native Debug.” [Online]. Available: <https://github.com/WebFreak001/code-debug>  
[10] Microsoft, “Debug Adapter Protocol.” [Online]. Available: <https://microsoft.github.io/debug-adapter-protocol>  
[11] Y. Wu, “rCore-Tutorial-v3.” [Online]. Available: <https://github.com/rcore-os/rCore-Tutorial-v3>  
[12] R. Cox, M. F. Kaashoek, and R. Morris, “Xv6, a simple Unix-like teaching operating system,” Sept. 5, 2013. [Online]. Available: <http://pdos.csail.mit.edu/6.828/2012/xv6.html>  
[13] Y. Zheng, “Starry-OS.” [Online]. Available: <https://github.com/Starry-OS/Starry>  