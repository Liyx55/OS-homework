# lab2
## 【实验题目】
实验 3 物理内存管理
## 【实验目的】
• 理解基于段页式内存地址的转换机制  
• 理解页表的建立和使用方法  
• 理解物理内存的管理方法  
## 【实验要求】
• 练习 0 ：填写已有实验  
• 练习 1 ：实现 first fit 连续物理内存分配算法（需要编程）  
• 练习 2 ：实现寻找虚拟地址对应的页表项（需要编程  
• 练习 3 ：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程  
## 【实验方案】 
 环境是ubuntu14，使用的工具主要是 sublime-text3,VSCode  
### 练习 0：
因为代码量较少，直接复制过即可。 
### 练习 3：  

由于一个物理页可能被映射到不同的虚拟地址上去（譬如一块内存在不同进程间共享），当这个页需要在一个地址上解除映射时，操作系统不能直接把这个页回收，而是要先看看它还有没有映射到别的虚拟地址上。这是通过查找管理该物理页的Page数据结构的成员变量ref（用来表示虚拟页到物理页的映射关系的个数）来实现的，如果ref为0了，表示没有虚拟页到物理页的映射关系了，就可以把这个物理页给回收了，从而这个物理页是free的了，可以再被分配。page_insert函数将物理页映射在了页表上。可参看page_insert函数的实现来了解ucore内核是如何维护这个变量的。当不需要再访问这块虚拟地址时，可以把这块物理页回收并在将来用在其他地方。取消映射由page_remove来做，这其实是page_insert的逆操作。
练习 3 的目的是要取消一个页的页表映射，当其引用次数为 0 时，也需要将此表释放，否则，只能释放页表入口。  
因此只需要先判断页表是否存在，然后对页表的引用减一，再判断是否需要释放该页即可。理解page_remove_pte函数中的注释,主要是根据注释来编写代码，需要补全在 kern/mm/pmm.c中的page_remove_pte函数。page_remove_pte函数的调用关系图如下所示： 
![](picture\lab2-3.1.png)	
•PTE_P，练习二中已经了解到 PTE_P用于表示page dir是否存在
```c
#define PTE_P           0x001                   // Present
``` 
•tlb_invalidate()函数
```c
// invalidate a TLB entry, but only if the page tables being
// edited are the ones currently in use by the processor.
void tlb_invalidate(pde_t *pgdir, uintptr_t la) {
    if (rcr3() == PADDR(pgdir)) { 
        invlpg((void *)la);
    }
}
```
当修改的页表是进程正在使用的那些页表，使之无效。

•page_ref_dec()函数：减少该页的引用次数，返回剩下的引用次数。
```c
static inline int page_ref_dec(struct Page *page) {
    page->ref -= 1; //引用数减一
    return page->ref;
}
```
•page_remove_pte()函数
```c
static inline void page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
	if (*ptep & PTE_P) {  //页表项存在
	        struct Page *page = pte2page(*ptep); //找到页表项
	        if (page_ref_dec(page) == 0) {  //只被当前进程引用
	            free_page(page); //释放页
	        }
	        *ptep = 0; //该页目录项清零
	        tlb_invalidate(pgdir, la); 
	        //修改的页表是进程正在使用的那些页表，使之无效
	    }
}
```
回答如下问题：

•数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

————存在对应关系：当页目录项或页表项有效时，二者之间有对应关系。练习2的问题其实已经说明了一些情况，实际上，pages每一项记录一个物理页的信息，而每个页目录项记录一个页表的信息，每个页表项则记录一个物理页的信息。可以说，页目录项保存的物理页面地址（即某个页表）以及页表项保存的物理页面地址都对应于Page数组中的某一页。由于页表项中存放着对应的物理页的物理地址，因此可以通过这个物理地址来获取到对应到的Page数组的对应项，具体做法为将物理地址除以一个页的大小，然后乘上一个Page结构的大小获得偏移量，使用偏移量加上Page数组的基地址皆可以或得到对应Page项的地址；
每个Page表示一个4K的页帧，对应一个页表项

•如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 鼓励通过编程来具体完成这个问题。

————我们知道lab1中虚拟地址和物理地址便是相等的，而lab2我们通过多个步骤建立了虚拟地址到物理地址的映射，在链接内核时不加入0xC0000000的偏移即可实现逻辑地址等于物理地址
或者在分配页时对页表项内的基址加-0xC0000的偏移。故如果取消该映射即可完成目标：(根据"系统执行中地址映射的四个阶段"内容进行反向完成)
1.首先修改链接脚本，将内核起始虚拟地址改为 0x100000，具体只需要将tools/kernel.ld中的代码进行很小的修改即可：
```c
ENTRY(kern_init)

SECTIONS {
            /* Load the kernel at this address: "." means the current address */
            . = 0x100000;         //修改这里为0x1000000即可

            .text : {
                       *(.text .stub .text.* .gnu.linkonce.t.*)
            }

```
2.修改虚拟内存空间起始地址，即将偏移量改为0，具体只需要将tkern/mm/memlayout.h中的代码进行很小的修改即可：
```c
//在memlayout.h中将KERNBASE 0Xc0000000改为 0x0即可
#define KERNBASE   0x00000000         //将“0xC0000000”改为“0x00000000”
//修改虚拟地址基址 减去一个 0xC0000000 就等于物理地址
```

3.注释掉取消0~4M区域内存页映射的的代码，具体只需要将kern/mm/pmm.c中的代码进行很小的修改即可：
```c
//disable the map of virtual_addr 0~4M
// boot_pgdir[0] = 0;

//now the basic virtual memory map(see memalyout.h) is established.
//check the correctness of the basic virtual memory map.
// check_boot_pgdir();
```
————由于在完全启动了ucore之后，虚拟地址和线性地址相等，都等于物理地址加上0xc0000000，如果需要虚拟地址和物理地址相等，可以考虑更新gdt，更新段映射，使得virtual address = linear address - 0xc0000000，这样的话就可以实现virtual address = physical address；
## 【遇到的问题】
需要注意的是，需要把开启页表关闭，否则会报错，因为页表开启时认为偏移量不为0，故会报错。但实际上，此时虚拟地址已经等于物理地址，任务完成。
## 【lab2重要知识点】
空闲页表的双向链表存在形式  
First-fit 的实现方法    
使用页的回收与合并  
页表的查询   
页表的释放等 
分页机制：如下图，通过页目录索引和页表项索引找到相应的页表项，找到对应的页基址，加上偏移量得到物理地址
+--------10------+-------10-------+---------12----------+
| Page Directory |   Page Table   | Offset within Page  |
|      Index     |     Index      |                     |
+----------------+----------------+---------------------+
## 【实验过程】
![](picture\lab2-3.jpg)
