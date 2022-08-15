---
title: Operating System | Ucore-Lab4
date: 2022-07-21 20:22:21
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

- The context struct is used for saving registers for switching the running process
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

- The trapframe struct can be writen when interrupt occurred, cpu will reset to run the ISR and find the corresponding handler. Then the trapframe would be used to parse and handle the interrupt.
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

### Practice 2

#### Problem 1
The following codes had shown the processing of do_fork().
```
    //    1. call alloc_proc to allocate a proc_struct
    if ((proc = alloc_proc()) == NULL) {
        goto fork_out;
    }
    proc->parent = current;
    
    //    2. call setup_kstack to allocate a kernel stack for child process
    if (setup_kstack(proc) != 0) {
        goto bad_fork_cleanup_proc;
    }
    
    //    3. call copy_mm to dup OR share mm according clone_flag
    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }
    
    //    4. call copy_thread to setup tf & context in proc_struct
    copy_thread(proc, stack, tf);
    
    //    5. insert proc_struct into hash_list && proc_list
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        proc->pid = get_pid();
        hash_proc(proc);
        list_add(&proc_list, &(proc->list_link));
        nr_process ++;
    }
    local_intr_restore(intr_flag);
    
    //    6. call wakeup_proc to make the new child process RUNNABLE
    wakeup_proc(proc);

    //    7. set ret vaule using child proc's pid
    ret = proc->pid;
```
Notice: 
- Step 3 has decided the way to operating the memory, duplicate or use same one with its parent.

#### Problem 2
Ucore can assign an unique process id for each process control block. Step 5 in Prob.1 has shown us that it will do the assigning opertion with interrupt disabled by command ```local_intr_save(intr_flag)```, because of that, cpu will not be rescheduled to other threads or processes until ```local_intr_restore(intr_flag)``` called.

### Practice 3

#### Problem 1
In lab4, ucore created 2 processes: idle and init. 
Ucore created a pcb named ```idle``` in the final stages then it set current process to idle. Idle process is used to run in idle status or switch the running processes. Init process is used to print string. When the ucore running to cpu idle, the control will be handed to idle or another processes which is can be scheduled.

#### Problem 2
Codes of proc_run:
```
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
}
```
In these codes, ```local_intr_save(intr_flag)``` and ```local_intr_restore(intr_flag)``` disabled the interrupt to ensure cpu executing commands in an atomic status which means it can not be interrputted. Because of that, registers of cpu can be reset with security. 