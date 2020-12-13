# lab3  
## 【实验题目】
实验 3 虚拟内存管理  
## 【实验目的】  
1. 了解虚拟内存的 Page Fault 异常处理实现
2. 了解页替换算法在操作系统中的实现
## 【实验要求】
    练习 0 ：填写已有实验  
    练习 1 ：给未被映射的地址映射上物理页（需要编程  
    练习 2 ：补充完成基于 FIFO 的页面替换算法（需要编程）  

## 【实验方案】
**练习 0**  
这里使用的是系统自带的meld 工具进行整合，只需要选择两个文件夹，然后进行比较，在需要修改的文件中点击 Compare, 填充代码。  
![](picture/lab3-1.jpg)  
**练习 1**   
do_pgfault 函数是这次要填写的内容 

用于处理 page fault 产生的异常，其中异常主要分为三种，页不存在、页在外存、没有权限等。   
在函数中出现两个新出现的结构体  	mm_struct： 
```c
// the control struct for a set of vma using the same PDT
struct mm_struct {
    list_entry_t mmap_list;        // linear list link which sorted by start addr of vma
    struct vma_struct *mmap_cache; // current accessed vma, used for speed purpose
    pde_t *pgdir;                  // the PDT of these vma
    int map_count;                 // the count of these vma
    void *sm_priv;                   // the private data for swap manager
};
```  
很显然，它是同一个页表目录项下所有 vma 的控制单元，首先包含了一个链表节点指向 vma，一个充当 cache 的 vma_struct 指针，PDT 的指针，vma 数量以及swap manager。  
vm_struct  
```c
// the virtual continuous memory area(vma), [vm_start, vm_end), 
// addr belong to a vma means  vma.vm_start<= addr <vma.vm_end 
struct vma_struct {
    struct mm_struct *vm_mm; // the set of vma using the same PDT 
    uintptr_t vm_start;      // start addr of vma      
    uintptr_t vm_end;        // end addr of vma, not include the vm_end itself
    uint32_t vm_flags;       // flags of vma
    list_entry_t list_link;  // linear list link which sorted by start addr of vma
};

```
当启动分页机制以后，如果一条指令或数据的虚拟地址所对应的物理页框不在内存中或者访问的类型有错误（比如写一个只读页或用户态程序访问内核态的数据等），就会发生页访问异常。产生页访问异常的原因主要有：

###目标页帧不存在（页表项全为0，即该线性地址与物理地址尚未建立映射或者已经撤销)；

###相应的物理页帧不在内存中（页表项非空，但Present标志位=0，比如在swap分区或磁盘文件上)，这在本次实验中会出现，我们将在下面介绍换页机制实现时进一步讲解如何处理；

###不满足访问权限(此时页表项P标志=1，但低权限的程序试图访问高权限的地址空间，或者有程序试图写只读页面).

当出现上面情况之一，那么就会产生页面page fault（#PF）异常。
CPU会把产生异常的线性地址存储在CR2中，并且把表示页访问异常类型的值（简称页访问异常错误码，errorCode）保存在中断栈中。

[提示]页访问异常错误码有32位。位0为１表示对应物理页不存在；位１为１表示写异常（比如写了只读页；位２为１表示访问权限异常（比如用户态程序访问内核空间的数据）

[提示]　CR2是页故障线性地址寄存器，保存最后一次出现页故障的全32位线性地址。CR2用于发生页异常时报告出错信息。当发生页异常时，处理器把引起页异常的线性地址保存在CR2中。操作系统中对应的中断服务例程可以检查CR2的内容，从而查出线性地址空间中的哪个页引起本次异常。

ucore中do_pgfault函数是完成页访问异常处理的主要函数，它根据从CPU的控制寄存器CR2中获取的页访问异常的物理地址以及根据errorCode的错误类型来查找此地址是否在某个VMA的地址范围内以及是否满足正确的读写权限，如果在此范围内并且权限也正确，这认为这是一次合法访问，但没有建立虚实对应关系。所以需要分配一个空闲的内存页，并修改页表完成虚地址到物理地址的映射，刷新TLB，然后调用iret中断，返回到产生页访问异常的指令处重新执行此指令。如果该虚地址不在某VMA范围内，则认为是一次非法访问。

#具体补充代码：
```c
do_pgfault()
 ptep = get_pte(mm->pgdir, addr, 1); // 根据引发缺页异常的地址 去找到 地址所对应的 PTE 如果找不到 则创建一页表
    if (*ptep == 0) { // PTE 所指向的 物理页表地址 若不存在 则分配一物理页并将逻辑地址和物理地址作映射 (就是让 PTE 指向 物理页帧)
        if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            goto failed;
        }
    } else { // 如果 PTE 存在 说明此时 P 位为 0 该页被换出到外存中 需要将其换入内存
        if(swap_init_ok) { // 是否可以换入页面
            struct Page *page = NULL;
            ret = swap_in(mm, addr, &page); // 根据 PTE 找到 换出那页所在的硬盘地址 并将其从外存中换入
            if (ret != 0) {
                cprintf("swap_in in do_pgfault failedn");
                goto failed;
            }
            page_insert(mm->pgdir, page, addr, perm); // 建立虚拟地址和物理地址之间的对应关系(更新 PTE 因为 已经被换入到内存中了)
            swap_map_swappable(mm, addr, page, 0); // 使这一页可以置换
            page->pra_vaddr = addr; // 设置 这一页的虚拟地址
        }
```
表示一个连续的虚拟内存块，包含 PDT 指针，首地址，尾地址等。 
 
#challenge：
时钟页替换算法把各个页面组织成环形链表的形式，类似于一个钟的表面。然后把一个指针（简称当前指针）指向最老的那个页面，即最先进来的那个页面。另外，时钟算法需要在页表项（PTE）中设置了一位访问位来表示此页表项对应的页当前是否被访问过。当该页被访问时，CPU中的MMU硬件将把访问位置“1”。当操作系统需要淘汰页时，对当前指针指向的页所对应的页表项进行查询，如果访问位为“0”，则淘汰该页，如果该页被写过，则还要把它换出到硬盘上；如果访问位为“1”，则将该页表项的此位置“0”，继续访问下一个页。
即：沿着链表遍历，将最近未被访问且未被修改的页换出内存。
```c
 swap_fifo.c
struct swap_manager swap_manager_fifo = {
     .name            = "fifo swap manager",
     .init            = &_fifo_init,
     .init_mm         = &_fifo_init_mm,
     .tick_event      = &_fifo_tick_event,
     .map_swappable   = &_fifo_map_swappable,
     .set_unswappable = &_fifo_set_unswappable,
    //  .swap_out_victim = &_fifo_swap_out_victim,
     .swap_out_victim = &_extended_clock_swap_out_victim, // 将 选择换出的页的函数改掉
     .check_swap      = &_fifo_check_swap,
};
static int _extended_clock_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick) {
    list_entry_t *head = (list_entry_t*)mm->sm_priv;
    assert(head != NULL);
    assert(in_tick == 0);
    list_entry_t *le = head->prev;
    assert(head != le);

    int i; // 循环三次 寻找合适的置换页
    for (i = 0; i < 2; i++) {
        /* 第一次循环 寻找 没被访问过的 且 没被修改过的 同时将被访问过的页的 访问位 清 0
            第二次循环 依然是寻找 没被访问过的 且 没被修改过的 因为到了此次循环 访问位都被清 0 了 不存在被访问过的
            只需要找没被修改过的即可 同时将被修改过的页 修改位 清 0
            第三次循环 还是找 没被访问过 且 没被修改过的 此时 第一次循环 已经将所有访问位 清 0 了
             第二次循环 也已经将所有修改位清 0 了 故 在第三次循环 一定有 没被访问过 也没被修改过的 页
        */
        while (le != head) {
            struct Page *page = le2page(le, pra_page_link);            
            pte_t *ptep = get_pte(mm->pgdir, page->pra_vaddr, 0);

            if (!(*ptep & PTE_A) && !(*ptep & PTE_D)) { // 没被访问过 也没被修改过 
                list_del(le);
                *ptr_page = page;
                return 0;
            }
            if (i == 0) {
                *ptep &= 0xFFFFFFDF;
            } else if (i == 1) {
                *ptep &= 0xFFFFFFBF;
            }
            le = le->prev;
        }
        le = le->prev;
    }
}
```
为了调用extended clock算法，将swap.c中swap_init函数的

sm = &swap_manager_fifo;改成sm = &swap_manager_clock;