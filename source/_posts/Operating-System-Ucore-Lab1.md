---
title: Operating System | Ucore-Lab1
date: 2022-06-23 09:06:27
tags:
- ['Operating System']
---

## Practices & Conclusion

### Practice 1

#### Problem 1
By the code, we can see that Makefile use gcc to do the compile operation to the source code with some libs step by step and output some object(.o) files. And then it uses ld to link those .o files and produce an executable object(obj/bootblock.out). Then use sign tools to generate a standard bootclock.o. Finally, use dd to generate an ucore.img file.
- PS
    - The ld command, also called the linkage editor or binder, combines object files, archives, and import files into one output object file, resolving external references. It produces an executable object file that can be run.
    - dd is a command-line utility for Unix and Unix-like operating systems whose primary purpose is to convert and copy files.
- From web
    ```
    GCC
        -g  增加gdb的调试信息
        -Wall   显示告警信息
        -O2     优化处理 (有 0，1，2，3，0是不优化)
        -fno-builtin   只接受以"__"开头的内建函数
        -ggdb   让gcc为gdb生成比较丰富的调试信息
        -m32    编译32位程序
        -gstabs     此选项以stabs格式生成调试信息，但是不包括gdb调试信息
        -nostdinc   不在标准系统目录中搜索头文件，只在-l指定的目录中搜索
        -fstack-protector-all   启用堆栈保护，为所有函数插入保护代码
        -E  仅做预处理，不进行编译，汇编和链接
        -x c  指明使用的语言为C语言

    LDD Flags
        -nostdlib   不连接系统标准启动文件和标准库文件，只把指定的文件传递给连接器
        -m elf\_i386    使用elf_i386模拟器
        -N      把text和data节设置为可读写，同时取消数据节的页对齐，取消对共享库的链接
        -e func     以符号func的位置作为程序开始运行的位置
        -Ttext addr  是连接时将初始地址重定向为addr （若不注明此，则程序的起始地址为0）

    ```

#### Problem 2
We can get the check logic of the image from the tools/sign.c. It shows us that an image size should less than 510 bytes and end with 0x55, oxAA.

### Pratice 2 & 3
In those pratices, we used gdb to debug the bootblock. What we can see is, the cpu load the 0x7c00 for the first command. It does something init like register eax, ds, es, ss. Then, it would enable the A20 mode to be compatible with the low address(20bits) for using high address(like 32bits). If we don't get A20 mode enabled, the address higher than 1MB(1088kb) will be wrapped around to zero by default in real mode. Next, prepare the gdt and gdtdesc structure then use lgdt command to transport them to registers. Then set CR0 register (protected mode) to 1 so that we got protected mode enabled. After the registers under the protected mode prepared, at this time, we had finished processor preparation to execute code with 32-bit address. Next, we will set the ebp and esp registers so that can execute bootmain.c.

### Pratice 4

#### Problem 1
Wait for the disk ready, and then use the supported method of hardware like outb (outb(..., 0x20)) to send the command to read sectors. Use insl to read a sector from the io.

#### Problem 2
First, we use the readseg method to load the ELFHDR from the kernel image(disk) to the virtual address @va. Then we use the the info stored in the ELFHDR to load the program content to the @va. Then the bootloader will call the entry point of kernel from the ELF Header so that the kernel code will take over the hardware.

### Practice 5
In this practice, we can use ebp and eip cleverly to trace the stackframe step by step. What we should to do is getting the current ebp value. Its value contains the last function ebp address before it. We can also use this ebp value to get the function params(esp+2) and eip value (the address which can return to last function(esp+1)). We can trace by those registers layer by layer until reach the ebp=0(means the address which the bootloader start the kernel) or the STACKFRAME_DEPTH.
- PS
    - Function Stack is built from high address to low address, so that we can use esp+1 to get the last eip register value(the address which can return to last function to resume the processing)

### Practice 6

#### Problem 1
In ucore, we can find the gatedesc at /kern/mm/mmu.h.
```
/* Gate descriptors for interrupts and traps */
struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
};
```
According to the structure, we can know that an item consist of 64bit aka 8bytes.
The low 16-32 bits saved the segment selector and we can use this to query segdesc from the GDT so that we can get the segment base address.
