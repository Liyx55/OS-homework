# lab5
## 【实验目的】
    •	了解第一个用户进程创建过程
    •	了解系统调用框架的实现机制
    •	了解ucore如何实现系统调用sys_fork/sys_exec/sys_exit/sys_wait来进行进程管理  

## 【实验要求】
    •	为了实现实验的目标，实验提供了3个基本练习和2个扩展练习，要求完成实验报告。
    •	练习0：填写已有实验
    •	练习1：加载应用程序并执行（需要编码）
    •	练习2：父进程复制自己的内存空间给子进程（需要编码）
    •	练习3：阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）
    •	选做
    •	扩展练习Challenge： ：实现 Copy on Write （COW）机制  

## 【实验方案】 
**代码修改**：
Lab5要求修改更新部分之前的代码  
**idt_init**：
```c
	extern uintptr_t __vectors[];	//声明中断入口 
	int i = 0;
	for (i = 0; i < (sizeof(idt) / sizeof(struct gatedesc)); i++) {
		SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);//为中断设置内核态权限
	}
	SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);//为T_SYSCALL设置用户态权限——增加的一行代码主要是设置相应的中断门
	lidt(&idt_pd);	//转入IDT
```

设置一个特定中断号的中断门，专门用于用户进程访问系统调用。在上述代码中，可以看到在执行加载中断描述符表 lidt 指令前，专门设置了一个特殊的中断描述符 idt[T_SYSCALL]，它的特权级设置为 DPL_USER，中断向量处理地址在 __vectors[T_SYSCALL] 处。这样建立好这个中断描述符后，一旦用户进程执行 INT T_SYSCALL 后，由于此中断允许用户态进程产生（它的特权级设置为 DPL_USER），所以 CPU 就会从用户态切换到内核态，保存相关寄存器，并跳转到 __vectors[T_SYSCALL] 处开始执行，


修改的代码为
```c  
SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
```  
**trap_dispatch**:
```c
        ticks++;		//ticks自增
		if (ticks % TICK_NUM == 0) {
			//print_ticks();
			assert(current != NULL);
			current->need_resched = 1;//时间片用完设置为需要调度
			//说明当前进程的时间片已经用完了。
		}
```
修改的代码为：
```c
			assert(current != NULL);
			current->need_resched = 1;
```
作用是说明当前进程的时间片已用完，需要重新调度。print_ticks必须去掉，否则测试时会报错。  
**alloc_proc**:
我们在原来的实验基础上，新增了 2 行代码：
```c
proc->wait_state = 0;//PCB 进程控制块中新增的条目，初始化进程等待状态  
proc->cptr = proc->optr = proc->yptr = NULL;//进程相关指针初始化
```
这两行代码主要是初始化进程等待状态、和进程的相关指针，例如父进程、子进程、同胞等等。新增的几个 proc 指针给出相关的解释如下：

```c
    parent:           proc->parent  (proc is children)
	children:         proc->cptr    (proc is parent)
	older sibling:    proc->optr    (proc is younger sibling)
	younger sibling:  proc->yptr    (proc is older sibling)
```
作用主要有两点，一是设置进程为等待态，二是设置进程的兄弟父母节点为空。  
**do_fork**：
```c
	assert(current->wait_state == 0); //确保当前进程正在等待
	set_links(proc); //将原来简单的计数改成来执行 set_links 函数，从而实现设置进程的相关链接 
    //list_add(&proc_list,&(proc->list_link));//insert proc_struct into hash_list && proc_list
    //nr_process++;
    //et_links(proc);
```
第一行是为了确定当前的进程正在等待，我们在 `alloc_proc` 中初始化 `wait_state` 为0。第二行是将原来的计数换成了执行一个`set_links`函数，因为要涉及到进程的调度，所以简单的计数肯定是不行的。
作用是为proc建立联系并且插入`proc_list`,`set_links`是上面两句代码的改进版,因此上面的代码必须去掉,否则测试时会出错。
```c
static void set_links(struct proc_struct *proc) {
    list_add(&proc_list,&(proc->list_link));//进程加入进程链表
    proc->yptr = NULL; //当前进程的 younger sibling 为空
    if ((proc->optr = proc->parent->cptr) != NULL) {
        proc->optr->yptr = proc; //当前进程的 older sibling 为当前进程
    }
    proc->parent->cptr = proc; //父进程的子进程为当前进程
    nr_process ++; //进程数加一
}
```  
可以看出，`set_links` 函数的作用是设置当前进程的 process relations。

### **练习2**:  

调用过程：do_fork—>copy_mm–>dup_mmap–>copy_range

    •		创建子进程的函数do_fork在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过copy_range函数（位于kern/mm/pmm.c中）实现的，请补充copy_range的实现，确保能够正确执行。
    •	请在实验报告中简要说明如何设计实现”Copy on Write 机制“，给出概要设计，鼓励给出详细设计。

总的代码：
```c
/*
   file_path:kern/process/proc.c
*/
/* do_fork -     parent process for a new child process
 * @clone_flags: used to guide how to clone the child process
 * @stack:       the parent's user stack pointer. if stack==0, It means to fork a kernel thread.
 * @tf:          the trapframe info, which will be copied to child process's proc->tf
 */
int do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC; //尝试为进程分配内存
    struct proc_struct *proc; //定义新进程
    if (nr_process >= MAX_PROCESS) { //分配进程数大于 4096，返回
        goto fork_out;  //返回
    }
    ret = -E_NO_MEM;  //因内存不足而分配失败
if ((proc = alloc_proc()) == NULL) { //分配内存失败
        goto fork_out; //返回
    }
    proc->parent = current; //设置父进程名字
    if (setup_kstack(proc) != 0) {//为进程分配一个内核栈
        goto bad_fork_cleanup_proc; //返回
    }
    if (copy_mm(clone_flags, proc) != 0) { //复制父进程内存信息
        goto bad_fork_cleanup_kstack; //返回
    }
    copy_thread(proc, stack, tf); //复制中断帧和上下文信息
    bool intr_flag; 
    //将新进程添加到进程的 hash 列表中
    local_intr_save(intr_flag);  //屏蔽中断，intr_flag 置为 1
    {
        proc->pid = get_pid(); //获取当前进程 PID
        hash_proc(proc);  //建立 hash 映射
        list_add(&proc_list,&(proc->list_link));//将进程加入到进程链表中
        nr_process ++;  //进程数加一
    }
    local_intr_restore(intr_flag); //恢复中断
    wakeup_proc(proc); //一切就绪，唤醒新进程
    ret = proc->pid; //返回当前进程的 PID
fork_out:  //已分配进程数大于4096
    return ret;
bad_fork_cleanup_kstack: //分配内核栈失败
    put_kstack(proc);
bad_fork_cleanup_proc: 
    kfree(proc);
    goto fork_out;
}
```

（1）. 填写代码需要用到两个新学到的函数page2kva和page_insert 
 
**page2kva**：

```c
static inline void* page2kva(struct Page* page) {
	return KADDR(page2pa(page));
}
```
其作用是返回一个page的kernel vritual addr，返回值是void指针。  

**page_insert**：

```c
int page_insert(pde_t * pgdir, struct Page* page, uintptr_t la, uint32_t perm) {
	pte_t* ptep = get_pte(pgdir, la, 1);
	if (ptep == NULL) {
		return -E_NO_MEM;
	}
	page_ref_inc(page);
	if (*ptep & PTE_P) {
		struct Page* p = pte2page(*ptep);
		if (p == page) {
			page_ref_dec(page);
		}
		else {
			page_remove_pte(pgdir, la, ptep);
		}
	}
	*ptep = page2pa(page) | PTE_P | perm;
	tlb_invalidate(pgdir, la);
	return 0;
}
```
学习了这两个函数后就可以根据注释编写代码了。

copy_mm：这个函数在Lab4中并没有实现
```c
// copy_mm - process "proc" duplicate OR share process "current"'s mm according clone_flags
//         - if clone_flags & CLONE_VM, then "share" ; else "duplicate"
//file_path:kern/process/proc.c
static int
copy_mm(uint32_t clone_flags, struct proc_struct *proc) {
    struct mm_struct *mm, *oldmm = current->mm;

    /* current is a kernel thread */
    if (oldmm == NULL) {  //当前进程地址空间为NULL，则复制失败，返回0
        return 0;
    }
    if (clone_flags & CLONE_VM) {  //如果共享内存的标记位为真，则可以共享内存
        mm = oldmm;//共享地址空间
        goto good_mm;
    }

    int ret = -E_NO_MEM;
    if ((mm = mm_create()) == NULL) {  //如果创建地址空间失败，报错
        goto bad_mm;
    }
    if (setup_pgdir(mm) != 0) {  //如果创建页表失败，报错
        goto bad_pgdir_cleanup_mm;
    }
		//打开互斥锁(用于避免多个进程同时访问内存)
    lock_mm(oldmm);
    {
        ret = dup_mmap(mm, oldmm);  //调用了dup_mmap函数
    }
    unlock_mm(oldmm);  //关闭互斥锁

    if (ret != 0) {
        goto bad_dup_cleanup_mmap;
    }

good_mm:
    mm_count_inc(mm); //共享地址空间的进程数加一
    proc->mm = mm;//复制空间地址
    proc->cr3 = PADDR(mm->pgdir); //复制页表地址
    return 0;
bad_dup_cleanup_mmap:
    exit_mmap(mm);
    put_pgdir(mm);
bad_pgdir_cleanup_mm:
    mm_destroy(mm);
bad_mm:
    return ret;
}

```

dup_mmap：file_path:kern/mm/vmm.c
```c
int
dup_mmap(struct mm_struct *to, struct mm_struct *from) {
    assert(to != NULL && from != NULL);  //保证非空，确保to和from创建成功
    list_entry_t *list = &(from->mmap_list), *le = list;
    //mmap_list 为虚拟地址空间的首地址,获取from的首地址
    while ((le = list_prev(le)) != list) {  //遍历所有段
        struct vma_struct *vma, *nvma;
        vma = le2vma(le, list_link);  //获取某一段的信息,并创建到新进程中。
        nvma = vma_create(vma->vm_start, vma->vm_end, vma->vm_flags);
        if (nvma == NULL) {
            return -E_NO_MEM;
        }

        insert_vma_struct(to, nvma); //向新进程插入新创建的段

        bool share = 0;
        //调用 copy_range 函数
        if (copy_range(to->pgdir, from->pgdir, vma->vm_start, vma->vm_end, share) != 0) {
            return -E_NO_MEM;
        }
    }
    return 0;
}

```
```c
/* copy_range - copy content of memory (start, end) of one process A to another process B
 * @to:    the addr of process B's Page Directory
 * @from:  the addr of process A's Page Directory
 * @share: flags to indicate to dup OR share. We just use dup method, so it didn't be used.
 *file_path:kern/mm/pmm.c
 * CALL GRAPH: copy_mm-->dup_mmap-->copy_range
 */
int
copy_range(pde_t *to, pde_t *from, uintptr_t start, uintptr_t end, bool share) {
    assert(start % PGSIZE == 0 && end % PGSIZE == 0);
    assert(USER_ACCESS(start, end));
    // copy content by page unit.
    do {
        //call get_pte to find process A's pte according to the addr start
        pte_t *ptep = get_pte(from, start, 0), *nptep;
        //获取页表内容
        if (ptep == NULL) {
            start = ROUNDDOWN(start + PTSIZE, PTSIZE);
            continue ;
        }
        //call get_pte to find process B's pte according to the addr start. If pte is NULL, just alloc a PT
        if (*ptep & PTE_P) {
            if ((nptep = get_pte(to, start, 1)) == NULL) {
                return -E_NO_MEM;
            }
        uint32_t perm = (*ptep & PTE_USER);
        //从ptep中获取页表的值
        struct Page *page = pte2page(*ptep);
        // alloc a page for process B
        struct Page *npage=alloc_page();
        assert(page!=NULL);
        assert(npage!=NULL);
        int ret=0;
        /* LAB5:EXERCISE2 YOUR CODE
         * replicate content of page to npage, build the map of phy addr of nage with the linear addr start
         *
         * Some Useful MACROs and DEFINEs, you can use them in below implementation.
         * MACROs or Functions:
         *    page2kva(struct Page *page): return the kernel vritual addr of memory which page managed (SEE pmm.h)
         *    page_insert: build the map of phy addr of an Page with the linear addr la
         *    memcpy: typical memory copy function
         *
         * (1) find src_kvaddr: the kernel virtual address of page
         * (2) find dst_kvaddr: the kernel virtual address of npage
         * (3) memory copy from src_kvaddr to dst_kvaddr, size is PGSIZE
         * (4) build the map of phy addr of  nage with the linear addr start
         *补充的代码在下面四行*/
        void * kva_src = page2kva(page);	//获取老页表的值(1)找到父进程需要复制的物理页在内核地址空间中的虚拟地址，这是由于这个函数执行的时候使用的时内核的地址空间
        void * kva_dst = page2kva(npage);	//获取新页表的值(2)找到子进程需要被填充的物理页的内核虚拟地址
        memcpy(kva_dst, kva_src, PGSIZE);	//复制(3)将父进程的物理页的内容复制到子进程中去
        ret = page_insert(to, npage, start, perm);
        //建立建立子进程页地址起始位置与物理地址的映射关系(4)
        assert(ret == 0);
        }
        start += PGSIZE;
    } while (start != 0 && start < end);
    return 0;
}

```
其原理是用memcpy将父进程的内核虚地址里的值复制给子进程的内核虚地址的空间。  

（2）. 创建子进程的时候不是将父进程的内存复制一份给子进程，而是将父进程的PDE复制给子进程，但是不允许子进程写入。当子进程请求写操作时，再给子进程分配一块空间，并替换其PDE中的项，才可以执行写操作。  

### 思考题：
>Copy-on-write（简称COW）的基本概念是指如果有多个使用者对一个资源A（比如内存块）进行读操作，则每个使用者只需获得一个指向同一个资源A的指针，就可以该资源了。若某使用者需要对这个资源A进行写操作，系统会对该资源进行拷贝操作，从而使得该“写操作”使用者获得一个该资源A的“私有”拷贝—资源B，可对资源B进行写操作。该“写操作”使用者对资源B的改变对于其他的使用者而言是不可见的，因为其他使用者看到的还是资源A。

即为进程执行fork系统调用进行复制的时候，父进程和子进程暂时共享相同的物理内存页；而当其中一个进程需要对内存进行修改的时候，再额外创建一个自己私有的物理内存页，将共享的内容复制过去，然后在自己的内存页中进行修改。

概要设计如下：

	•	do_fork：进行内存复制时，如在copy_range中,不进行复制,而是将父进程和子进程的虚拟页映射上同一个物理页,然后将父进程的PDE直接赋值给子进程，将PTE_W置为0（不可写入),但应用程序试图写入某个共享页就会产生页访问异常
	•	page_fault：在page_fault的ISR(中断处理函数)部分，增加对这种异常的处理，处理方法为将当前的共享页的内容复制过去,建立出错的线性地址与新创建的物理页面的映射关系，将PTE设置设置成非共享的；然后查询原先共享的物理页面是否还是由多个其他进程共享使用的，如果不是的话，就将对应的虚地址的PTE进行修改，删掉共享标记，恢复写标记;

