---
title: Operating System | Ucore-Lab3
date: 2021-07-17 19:00:21
tags:
- ['Operating System']
---

## Practices & Conclusion

### Practice 0
Use software - meld to merge the codes. ^-^, better than diff & patch command!

### Prcatice 1

#### Problem 1
First, we should know the consist of the Page Directory/Table Entry.
```
/* page table/directory entry flags */
#define PTE_P           0x001                   // Present
#define PTE_W           0x002                   // Writeable
#define PTE_U           0x004                   // User
#define PTE_PWT         0x008                   // Write-Through
#define PTE_PCD         0x010                   // Cache-Disable
#define PTE_A           0x020                   // Accessed
#define PTE_D           0x040                   // Dirty
#define PTE_PS          0x080                   // Page Size
#define PTE_MBZ         0x180                   // Bits must be zero
#define PTE_AVAIL       0xE00                   // Available for software use
                                                // The PTE_AVAIL bits aren't used by the kernel or interpreted by the
                                                // hardware, so user processes are allowed to set them arbitrarily.
```
Accroding to this description, when we want to check or alloc pt/p of pd/pt, we can easily know the present, writeable, user flags, etc. So that we can use less steps to support those operations on pd/pt.

#### Problem 2
When page fault raised, some handling operations for exceptions should be done by the hardwares. Cpu will save the exception linear address to the CR2 register and backup the current context in the kernel stack (push the EFLAGS, CS, EIP, errCode). Then load the interrupt service address to cs and eip to run the corresponding handling services. After the fault handled, cpu need to restore the registers and exception application context.

### Prcatice 2
The current swap manager can support the extended clock ALGO.

1. Extended clock ALGO.

    It can be implemented by adding a "second chance" bit to each memory frame-every time the frame is considered (due to a reference made to the page inside it), this bit is set to 1, which gives the page a second chance, as when we consider the candidate page for replacement, we replace the first one with this bit set to 0 (while zeroing out bits of the other pages we see in the process). Thus, a page with the "second chance" bit set to 1 is never replaced during the first consideration and will only be replaced if all the other pages deserve a second chance too!

2. Timing to swap-in or swap-out

    We can use PTE to record the second bit (dirty bit) and differ those pages. Swap out had two different way to achieve, positive and negative way. The positive way means that operating system will check the page table periodly and do the swap in or out operations, it can be implemented by the clock interrup. The negative way corresponds that swap in or out the target page when using. What we should notice is, when doing swap-out operations, we need to swap it out to the disk directly or record the changes when a dirty page (which is changed in memory) to merge several changes.