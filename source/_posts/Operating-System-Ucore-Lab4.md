---
title: Operating System | Ucore-Lab4
date: 2022-07-17 19:00:21
tags:
- ['Operating System']
---

## Practices & Conclusion

### Practice 1
- The process control block in ucore is designed like this
    ```
    struct proc_struct {
        enum proc_state state;                      // Process state
        int pid;                                    // Process ID
        int runs;                                   // the running times of Proces
        uintptr_t kstack;                           // Process kernel stack
        volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
        struct proc_struct *parent;                 // the parent process
        struct mm_struct *mm;                       // Process's memory management field
        struct context context;                     // Switch here to run process
        struct trapframe *tf;                       // Trap frame for current interrupt
        uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
        uint32_t flags;                             // Process flag
        char name[PROC_NAME_LEN + 1];               // Process name
        list_entry_t list_link;                     // Process link list 
        list_entry_t hash_link;                     // Process hash list
    };
    ```

- The context struct is used for saving registers for switching the running process.
The context and trapframe struct of pcb are designed like this:
    ```
    struct context {
        uint32_t eip;
        uint32_t esp;
        uint32_t ebx;
        uint32_t ecx;
        uint32_t edx;
        uint32_t esi;
        uint32_t edi;
        uint32_t ebp;
    };
    ```

- The trapframe struct can be writen when interrupt occurred, cpu will run the ISR to find the corresponding handler. Then the trapframe would be used to parse and handle the interrupt.
    ```
    struct trapframe {
        struct pushregs tf_regs;
        uint16_t tf_gs;
        uint16_t tf_padding0;
        uint16_t tf_fs;
        uint16_t tf_padding1;
        uint16_t tf_es;
        uint16_t tf_padding2;
        uint16_t tf_ds;
        uint16_t tf_padding3;
        uint32_t tf_trapno;
        /* below here defined by x86 hardware */
        uint32_t tf_err;
        uintptr_t tf_eip;
        uint16_t tf_cs;
        uint16_t tf_padding4;
        uint32_t tf_eflags;
        /* below here only when crossing rings, such as from user to kernel */
        uintptr_t tf_esp;
        uint16_t tf_ss;
        uint16_t tf_padding5;
    } __attribute__((packed));
    ```
