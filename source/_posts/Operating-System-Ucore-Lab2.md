---
title: Operating System | Ucore-Lab2
date: 2022-07-06 02:05:40
tags:
- ['Operating System']
---

## Above all
1. First we should notice is a 32-bit register can store 2^32 different RAM addresses. Each RAM address corresponds to a Byte(8 bits) in modern RAMs. So a 32-bit register can corresponds (2^32 * 2^3) which equals 4GB.
2. What we should know second is, before the kernal started, the bootloader had enable the a20 and protected segment mode. After that we got a mapping (VA = LA = PA) with base addr 0x0. 
3. In kernel.ld, it defined the new base, data, bss addr(VA) under the segment mode. 
4. After loading kernel.ld and executing the entry.S had modified the based addr to 0xC0000000 in GDT. And tune the kernel start linked addr to virtual address(VA) 0xC0100000. Of course, this VA(0xC0100000) mapping to the PA (0x00100000 = 2^2^5 RAM addresses, 1MB size in RAM memory). We had achieved the mapping (VA|LA - 0xC0000000 = PA).
5. Mapping and init the pages descriptors and the physical address. Now, we can use alloc and free opertions of pages.
6. Alloc page descriptors for boot_pgdir corresponds 4M memory. Then use some tricks to map pages-segments and fill the boot_pgdir[PDX(KERNBASE)] coresponds to itself. Finally, refresh the cr3(pgdir) enable the cr0(paging flag) and realod the gdt.
6. Reset the boot pgdir(corresponds to the boot usage), cr3 register(used to store the pgdir), then remap the segment
7. Virtual memory map of Ucore
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
    *                            |   Cur. Page Table (Kern, RW)    | RW/-- PTSIZE  -- use 4KB to store 1M Page Descritor(4B) => 1M * Page Frame(4KB) = 4G Memory
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
8. Segmented Paging Memory Management
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
In this practice, we achieved the first fit memory alloc ALGO. What we should notice is diffing 
- physical address of RAM(memory)
- Page descriptor (about 4B, not physical page frame(4KB), one physical frame can contain 1M Page descriptor) which manage those physical address, stored at area upon the VPT. 
- free_list, corresponds a mutable list of usable page_link. 
We can use page_link of free_list to get the page descriptor with function le2page(). The alloc and free operations correspond to modifying the free_list and setting the flags and property of the pages. As for first fit memory alloc algorithm. It asks us to find first continuous free pages from the free_list to alloc. We need to pay more attention to keep the free_list order by addresses to keep them continuous when doing init, alloc and free operations.

### Practice 2

#### Problem 1
A linear address 'la' has a three-part structure as follows:
```
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |     Index      |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \----------- PPN(la) -----------/
```
- Page Directory Entry (use high 10 bits to get)
```
pde = (pte_base_addr & ~0x0FFF) | PTE_U | PTE_W | PTE_P
```
- Page Table Entry (use mid 10 bits to get)
```
pte = (physical_address  & ~0x0FFF) | PTE_P | PTE_W
```
In ucore, we would need la and those two struct to get an physical address. What's more important is we can change the pte value to change the mapping of the virtual address and physical address(physical page frame).

#### Problem 2
When the processor visit an nonexistent page, a page fault(#PF) will happen. An interrupt will be triggered, then the interrupt handling service will be pushed to the stack and executed. If the visit is valid then the service will alloc a new page for the visit and refresh the TLB. Finally, call the iret to interrupt and jump back to the exception address and go on process.

### Practice 3

#### Problem 1
There is a mapping relation existed. We can use pa stored in pte to get the page descriptor.

#### Problem 2
We can refresh the gdt to make va = la - 0xC0000000.