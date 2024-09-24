---
title: Syscalls on Windows 11
---

# Reversing the Windows 11 Syscall Handler

Author: Andrea Di Dio

If you have any further questions or suggestions after reading this writeup feel free to contact me at a.didio@student.vu.nl or on Twitter ([@hammertux](https://twitter.com/hammertux "hammertux")). I will try to answer any questions or adopt any suggestions :)

## Introduction

The aim of this post is to briefly explain the kernel routines involved in a Windows system call (in Windows terminology, a _system service_). In order to retrieve such information I used [Ghidra](https://ghidra-sre.org/) in order to reverse engineer the relevant parts of the `ntoskrnl.exe` binary and [WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools) in order to step through the kernel execution upon a system service.

## Syscalls (aka System Services)

### Relevant Components

- `ntdll.dll`: special system support library which resides in _user-space_ and offers system service dispatch stubs to Windows executive system services.
- Relevant MSRs:
  - `STAR` (0xc0000081)
  - `LSTAR` (0xc0000082)
  - `SFMASK` (0xc0000084)
- Kernel Processor Control Region (`KPCR`): Contains information about the **current processor**. Each processor has its own KPCR. Contains plenty of metadata including a pointer (`Prcb`) to a `KPRCB` (see below).
- Kernel Processor Control Block (`KPRCB`): Massive structure which contains more information about the state of a processor. Notably, it holds a pointer (`CurrentThread`) which points to a `KTHREAD` structure that olds the state of the current running kernel thread. The offset of this member is hardcoded to be `KPCR + 0x188`.

### The syscall life-cycle

#### Syscall Dispatch:

1. User issues call to a specific syscall (e.g., `NtCreateFile()`).
2. This invokes the `ntdll!Nt*` stub which stores the corresponding syscall number in `EAX` (for `NtCreateFile()` this is `0x55`) and executes the `syscall` instruction to trap into the kernel.
3. When the syscall instruction is executed:
   1. The Code Segment (`CS`) is loaded from Bits 32 to 47 in `STAR`, which Windows sets to 0x0010 (`KGDT64_R0_CODE`).
   2. The Stack Segment (`SS`) is loaded from Bits 32 to 47 in `STAR` + 8, which gives us 0x0018 (`KGDT_R0_DATA`).
   3. The Program Counter (`RIP`) is saved in `RCX` and the new value (i.e., the address of the syscall handler) is loaded from `LSTAR`. This either resolves to the `nt!KiSystemCall64` or `nt!KiSystemCall64Shadow` functions.
   4. The processor flags (`RFLAGS`) are saved in `R11` and then masked with `SFMASK`, which Windows sets to 0x4700 (Trap Flag, Direction Flag, Interrupt Flag, and Nested Task Flag).
   5. The Stack Pointer (`RSP`) and all other segments (`DS`, `ES`, `FS`, and `GS`) are kept to their current user-space values.
4. In the `nt!KiSystemCall64` syscall handler, after the `swapgs` instruction, the `GS` points to the Kernel Processor Control Region (`KPCR`).
5. The current `RSP` is saved into the `UserRsp` field of the `KPCR`
6. The new stack pointer is loaded from the `RspBase` field of the `KPRCB`
7. Now that the kernel stack is loaded, the function builds a _trap frame_. This includes storing in the frame the `SegSs` set to `KGDT_R3_DATA (0x2B)`, Rsp from the UserRsp in the PCR, EFlags from R11, SegCs set to KGDT_R3_CODE(0x33), and storing Rip from RCX.
8. Load `RCX` from `R10` to comply with the x64 ABI which requires the first argument of any function to be in `RCX`.
9. Flush _uarch buffers_ and temporarily disable SMAP with the `stac` instruction.

#### Returning to Userland:

#### Calls:

- `nt!KiSystemCall64`
  - Sets up the kernel stack and executes the `stac` instruction to disable SMAP.
- `nt!KiSystemServiceUser`
  - Retrieves `KTHREAD` structure's address from the `GS` register and stores it in `RBX`.
  - Sets the `FirstArgument` (from `RCX`) and `SystemCallNumber` (from `EAX`) fields of the `KTHREAD` structure.
- `nt!KiSystemServiceStart`
  - Carves out of the syscall number two fields: `Table Identifier` (bits [12-13]) and the `System Call Number` (bits [0-11]). The first can only have the value 00 (for native syscalls i.e., those coming from `ntdll.dll`) or 01 (for GUI functions i.e., those coming from `win32u.dll`) as there are two 'syscall tables'. After this function, the table identifier is in `EDI` and the _true_ syscall number in `EAX`.
- `nt!KiSystemServiceRepeat`
  - Loads in `R10` and `R11` two `Service Descriptor Tables` (`SDT`) namely, the `nt!KeServiceDescriptorTable` and the `nt!KeServiceDescriptorTableShadow`. The first contains the `System Service Table` a.k.a `System Service Dispatch Table` --- `SSDT` (`nt!KiServiceTable`) for native system calls while the latter holds the same table **plus** the System Service Table for GUI functions (`win32k!W32pServiceTable`).
  - Check `RBX` (where the _current_ `KTHREAD` is stored) to see if the `GuiThread` bit is set in the flag member of `KTHREAD + 0x78`. (`test dword ptr [RBX + 0x78], 80h`). Note that on the first call to this function, the `GuiThread` flag is not set as the kernel cannot know whether the current thread is executing a GUI function or not.
  - If the `GuiThread` flag is not set, i.e., a native syscall has been issued from a thread, the code jumps to a check to see if `EAX` is above the address `[R10 + RDI + 0x10]` (i.e., System Call Number > [`nt!KeServiceDescriptorTable` + Table Identifier + 0x10]). This essentially check that the System Call Number is _below_ the `nt!KiServiceTable->ServiceLimit`.
  - If the check fails (i.e., the syscall number is above the possible range), the code checks to see if `EDI == 0x20` to check if the function is a GUI function. If it is, the thread is converted to a GUI thread (`nt!KiConvertGuiThread`) and the code jumps back to the start of `nt!KiSystemServiceRepeat`. If `EDI != 0x20`, the syscall number is out of range and the routine exits the syscall processing.
  - In the case of a GUI thread, the code checks for another flag in the `KTHREAD` structure (`test dword ptr [RBX + 0x78], 200000h`) namely, the `RestrictedGuiThread` flag. If this flag is set, `nt!KeServiceDescriptorTableFilter` SSDT is loaded in `R10`. If not, the normal GUI SSDT is loaded (`nt!KeServiceDescriptorTableShadow`).
  - After this code block, `R10` is incremented by `RDI` (Table Identifier) meaning that it will hold either `nt!KeServiceDescriptorTable + 0x00` (i.e., `nt!KiServiceTable`) for native syscalls, or `nt!KeServiceDescriptorTableShadow + 0x20` (i.e., `win32k!W32pServiceTable`)for GUI functions (disregarding win32k filters for simplicity).
  - The entry in the SSDT for the corresponding System Call Number is loaded in `R11` with the instruction `movsxd R11, dword ptr [R10 + RAX * 0x4]` i.e., `R11 = SSDT + System Call Number * 4` where the multiplication by 4 is needed as each entry in the SSDT is 4 bytes long. Each entry in the SSDT contains two values. The lowest order nibble (bits [0-3]) holds the number of arguments for the function passed via the **stack** (Following x64 convention, the first 4 arguments are passed via registers `RCX`, `RDX`, `R8` and, `R9` meaning that this part of the SSDT entry is 0 for all syscalls with _at most_ 4 arguments). The rest of the entry (bits [4-31]) contain the Relative Value Address (`RVA`) in the SSDT of the correct native function.
  - At this point, the last operation this function executes is to add the RVA of the function to the correct SSDT which resolves to the address of the internal routine corresponding to the syscall issued in userland. (`add R10, R11`). If the function is a GUI function, the last step is to load the Thread Execution Environment structure (`TEB`) in `R11` with the following instruction `mov R11,qword ptr [RBX + 0xf0]`. If we are dealing with a normal NT syscall we jump in the middle of the next function (`nt!KiSystemServiceGdiTebAccess`) to skip some GUI related checks.
- `nt!KiSystemServiceGdiTebAccess`
  - In the case of a GUI function, at the start of the function, there is a check to see if the `Teb->GDIBatchCount` is 0 or not (`cmp dword ptr [R11 + 0x1740],0x0`). If this field is 0, it skips the next 19 instructions and jumps to the same point as for a normal NT syscall.
  - Following the code path for normal NT syscalls, the function will once again check to see if the syscall has arguments on the stack `and EAX, 0xF`. If there are no arguments passed on the stack for the syscall, the function jumps directly to `nt!KiSystemServiceCopyEnd`. On the other hand, if the syscall has arguments, the code makes space on the stack for the maximum number of arguments, multiplies the number of arguments by 8 (`shl EAX, 0x3`).
  - Check if the provided arguments are in the address range reserved for userspace by comparing `RSI` to `nt!MMUserProbeAddress`.
  - Finally, the code loads the address of the `nt!KiSystemServiceCopyEnd` routine into `R11`. Subtracts `EAX` (number of args * 8) from `R11`. This is done because the `nt!KiSystemServiceCopyStart` routine precedes the `nt!KiSystemServiceCopyEnd` routine and only contains `mov` instructions which are 4 bytes each. For each argument two `mov` instructions are needed to copy the user data in kernel space. The code then jumps to the value stored in `R11`.
- `nt!KiSystemServiceCopyStart`
  - This function is only ever called when the system call has arguments passed on the **stack**. I.e., the code in this function is only executed if the syscall being executed has more than 4 arguments.
  - The sole purpose of this function is to copy the arguments from the user space stack (in `RSI`) to the kernel space buffer (in `RDI`).
- `nt!KiSystemServiceCopyEnd`
  - In this function, the kernel system service routine is finally called, e.g., `nt!NtCreateFile()`. This is done by copying `R10` in `EAX` and jumping to `EAX`.
- `nt!KiSystemServiceExit`
  - Cleanup and `swapgs / sysret`

## WinDbg Flow:

### Tracing a function from a usermode application in the kernel

#### Setup:

- Compile the uspace binary without ASLR (`/DYNAMICBASE:NO`)
- Note down the address of the `main` function for the uspace binary. Can be found with some static analysis tool like ghidra or IDA.

In kd:

```
!gflag +ksl #loads kern symbols
sxe ld <uspace_binary> #breaks whenever the application <uspace_binary> is started
g #resume execution of target machine
```

Open the uspace_binary on the target machine. This will hit the breakpoint.

```
!gflag -ksl
!process -1 0 #Check that the current process is running
.process #Set the process to be the implicit process
bp <addr_main> #Set a breakpoint in the main function of the uspace_binary
g # Resume execution
```

