---
title: Operating System | Ucore-Lab2
date: 2021-07-06 02:05:40
tags:
- ['Operating System']
---

## Presupposition
1. First we should notice a 32-bits register can store 2^32 different RAM addresses. Each RAM address corresponds to a Byte(8 bits) in modern RAMs. So a 32-bit register can corresponds (2^32 * 2^3) which equals 4GB.
2. About the kernel boot and memory detect, init, map processing are as follow.
    1. What we should know secondly is, before the kernal started, the bootloader had enable the a20 and protected segment mode. After that we got a mapping (VA = LA = PA) with base addr 0x0. 
    2. In kernel.ld, it defined the new base, data, bss addrress under the segment mode by lgdt. 
    3. After loading kernel.ld and executing the entry.S had modified the based addr to 0xC0000000. They tune the kernel start linked addr to virtual address(VA) 0xC0100000. Of course, this VA(0xC0100000) mapping to the PA (0x00100000 = 1*(2^4)^5 = 2^20 RAM addresses, 2^20B = 1MB size in RAM memory). We had achieved the mapping (VA|LA = PA + 0xC0000000).
    4. Detect the physical memory layout by e820 then map and init pages descriptors and the free page list Now, we can use alloc and free opertions of pages.
    5. Alloc a page frame(4KB) for boot_pgdir(virtual address corresponds to this page frame) and use it to store the page table. 4KB can store 1024 ptes(32bit, 4B), which means 1024 * 4B = 4MB physical memory. Set its physical memory address to boot_pgdir[PDX(VPT)] which points itself in virtual page table.
    6. Remap the segment memory layout by boot_map_segment. Via mapping the pte and physical memory address, it maps all linear address (KERNBASE ~ KERNBASE + KMEMSIZE) to the physical address (0 ~ KMEMSIZE). Now we get a full size page table mapping to KMEMSIZE physical memory.
    7. ```boot_pgdir[0] = boot_pgdir[PDX(KERNBASE)]``` Set a temporary mapping for kernel code and data, which will be used by cpu low addr to high addr switch. It allocs just one pde means 2^10 * 4KB = 4MB for temporary memory so that kernel may be crash if it's not enough. The solution will be allocing more temporary memory for kernel info before switch.
    8. Refresh the cr3(pgdir) and cr0(paging flag) to enable the paging mode
    9. Init the TSS then realod the gdt.
        The task state segment (TSS) is a structure on x86-based computers which holds information about a task. It is used by the operating system kernel for task management. Specifically, the following information is stored in the TSS:

        - Processor register state
        - I/O port permissions
        - Inner-level stack pointers
        - Previous TSS link
    10. Cancel the temporary memory mapping of boot_pgdir[0] which used in cpu running address switches.
3. Page descriptor is using for describing the memory perporties. Page table is using for mapping the virtual and physical address.
4. Virtual memory map of Ucore is as follow
    ```
    /* *
    * Virtual memory map:                                          Permissions
    *                                                              kernel/user
    *
    *     4G ------------------> +---------------------------------+
    *                            |                                 |
    *                            |         Empty Memory (*)        |
    *                            |                                 |
    *                            +---------------------------------+ 0xFB000000
    *                            |   Cur. Page Table (Kern, RW)    | RW/-- PTSIZE  -- use 4KB => 2^10 pdes =>  2^20 ptes => 2^20 * Page Frame(4KB) = 4G Physical Memory
    *     VPT -----------------> +---------------------------------+ 0xFAC00000
    *                            |        Invalid Memory (*)       | --/--
    *     KERNTOP -------------> +---------------------------------+ 0xF8000000
    *                            |                                 |
    *                            |    Remapped Physical Memory     | RW/-- KMEMSIZE(896MB)
    *                            |                                 |
    *     KERNBASE ------------> +---------------------------------+ 0xC0000000
    *                            |                                 |
    *                            |                                 |
    *                            |                                 |
    *                            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    * (*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
    *     "Empty Memory" is normally unmapped, but user programs may map pages
    *     there if desired.
    *
    * */
    ```
5. Segmented Paging Memory Management
- ![Segmented Paging Memory Management](images/SegmentedPaging.png)

## Practices & Conclusion

### Practice 0
For achieving the lab2 better, we need to use diff and patch to merge the code from lab1.
- diff
    ```
    diff -Nu lab2/kern/debug/kdebug.c lab1/kern/debug/kdebug.c >> lab_merge.patch
    -N treat absent files as empty
    -u unified style, if your system lacks it or if recipient may not have it, use "-c"
    -r recursive, so do subdirectories
    ```
- patch
    ```
    patch -p0 < ../lab_merge.patch
    -s == silent except errors
    -p0 == needed to find the proper folder
    PS: use cd to origin path and use -p param to strip the path, like lab2/kern/debug/kdebug.c => -p1 = kern/debug/kdebug.c
    ```
- notes from web
    ```
    # to create patch:
    # copy <directory> backup to something like <directory>.orig alongside it
    cp -r <path_to>/<directory> <path_to>/<directory>.orig
    # create/update/delete files/folders in <directory> until desired state is reached
    # change working directory to <directory>
    cd <path_to>/<directory>
    # create patch file alongside <directory>
    diff -Naru ../<directory>.orig . > ../file.patch
    # -N --new-file Treat absent files as empty.
    # -a --text Treat all files as text.
    # -r --recursive Recursively compare any subdirectories found.
    # -u -U NUM --unified[=NUM] Output NUM (default 3) lines of unified context.

    # to apply patch:
    # change working directory to <directory>
    cd <path_to>/<directory>
    patch -s -p0 < <path_to>/file.patch
    # -s or --silent or --quiet Work silently, unless an error occurs.
    # -pN or --strip=N Strip smallest prefix containing num leading slashes from files.

    # to undo patch (note that directories created by patch must be removed manually):
    # change working directory to <directory>
    cd <path_to>/<directory>
    patch -Rs -p0 < <path_to>/file.patch
    # -R or --reverse Assume that patch was created with the old and new files swapped.
    # -s or --silent or --quiet Work silently, unless an error occurs.
    # -pN or --strip=N Strip smallest prefix containing num leading slashes from files.
    ```

### Practice 1
In this practice, we achieved the first fit memory alloc ALGO. What we should notice is diffing the follows.
- physical address of RAM(memory)
- page descriptor (size 4B, not page frame(4KB), one page frame can contain 2^10 Page descriptor) which record those pages info of physical addresses, stored at the address of ending of kernel. 
- free_list, corresponds a mutable list of usable page_link.

We can use page_link of free_list to get the page descriptor with function le2page(). The alloc and free operations correspond to modifying the free_list and setting the flags and property of the pages. 

As for first fit memory alloc algorithm, it asks us to find first continuous free pages from the free_list to alloc. We need to pay more attention to keep the free_list order by addresses and keep the mapping continuous in physical memory level when doing init, alloc and free operations.
    
- default_init: init a free pages list.
- default_init_memmap: create page descriptor for every usable PGSIZE physical memory block detected by e820 and link them to the free pages list.
- default_alloc_pages: find the first n continuous pages in free list, set some property, remove them from the free list and return the pointer points to the start page descriptor.
- default_free_pages: clear up the page properties and relink them to free list, the continuity of the physical memory should be noticed.

### Practice 2

#### Problem 1
First, the page directory entry corresponding to the virtual address range [VPT, VPT + PTSIZE) points to the page directory itself. Thus, the page directory is treated as a page table as well as a page directory.

Second, a linear address 'la' has a three-part structure as follows:
```
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |     Index      |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \----------- PPN(la) -----------/
```
According to that, we can get the pde/pte as follows:
- Page Directory Entry (use high 10 bits to get)
    ```
    pde = (pte_base_addr & ~0x0FFF) | PTE_U | PTE_W | PTE_P
    ```
- Page Table Entry (use mid 10 bits to get)
    ```
    pte = (physical_address  & ~0x0FFF) | PTE_P | PTE_W
    ```

In ucore, we would need la and those two struct to get an physical address. What's more important is we can change the pte value to change the mapping of the virtual address and physical address(start address of a page frame).

#### Problem 2
When the processor visit an nonexistent page, a page fault(#PF) will happen. An interrupt will be triggered, then the interrupt handling service will be pushed to the stack and executed. If the visit is valid then the service will alloc a new page for the visit and refresh the TLB. Finally, call the iret to interrupt and jump back to the exception address and go on process. If the visit is invalid, error will be raised.

### Practice 3

#### Problem 1
There is a mapping relation existed between page descriptors and pdes/ptes. The transition is like the follow:
```
    PTE => Page: use index which composited by high 20 bits of pte to get pages[index]
    &pages[(
        (
            (uintptr_t)(
                (
                    (uintptr_t)(pte) & ~0xFFF
                )
            )
        ) >> PTXSHIFT
    )]    
```

#### Problem 2
Reset KERNBASE from 从0xC0000000 to 0x00000000 will get the target which maps the VA=LA=PA