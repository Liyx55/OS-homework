# lab4
## 【实验题目】实验 5 内核线程管理 
## 【实验目的】  
了解内核线程创建/执行的管理过程  了解内核线程的切换和基本调度过程 
## 【实验要求】 
    •	为了实现实验的目标，实验提供了 3 个基本练习和 2 个扩展练习，要求完成实验报告。 
    •	练习 0：填写已有实验 
    •	练习 1：分配并初始化一个进程控制块（需要编码） 
    •	练习 2：为新创建的内核线程分配资源（需要编程） 
    •	练习 3：阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。 （无需编程） 
    •	选做 
    •	扩展练习 Challenge：实现支持任意大小的内存分配算法  
## 【实验内容】
lab2和lab3完成了对内存的虚拟化，但整个控制流还是一条线串行执行。lab4将在此基础上进行CPU的虚拟化，即让ucore实现分时共享CPU，实现多条控制流能够并发执行。从某种程度上，我们可以把控制流看作是一个内核线程。
实验2/3完成了物理和虚拟内存管理，这给创建内核线程（内核线程是一种特殊的进程）打下了提供内存管理的基础。当一个程序加载到内存中运行时，首先通过ucore OS的内存管理子系统分配合适的空间，然后就需要考虑如何分时使用CPU来“并发”执行多个程序，让每个运行的程序（这里用线程或进程表示）“感到”它们各自拥有“自己”的CPU。

本次实验将首先接触的是内核线程的管理。内核线程是一种特殊的进程，内核线程与用户进程的区别有两个：

内核线程只运行在内核态
用户进程会在在用户态和内核态交替运行
所有内核线程共用ucore内核内存空间，不需为每个内核线程维护单独的内存空间
而用户进程需要维护各自的用户内存空间
## 【实验方案】  
**练习 0：**   
 使用 meld 工具进行整合合并，对比 lab3 和 lab4 两个文件夹，对有区别的文件一一比较添加 lab3 的代码 
![](picture/lab4-1.jpg)
**练习 2：**  
do_folk 是为了给新创建的线程分配资源，按照注释和手册编写代码比较简单，但是有几个地方需要注意： 
1.	在函数为有几个标记，其中 bad_fork_cleanup_kstack，bad_fork_cleanup_proc 并未被用到，因此推测这些标记能被用于代码中。bad_fork_cleanup_proc，是释放线程，而 bad_fork_cleanup_kstack 则会对 kstack 也进行操作。因此在 setup_kstack 后的代码如果出错则需要跳转到 bad_fork_cleanup_kstack。 
2.	hash_list 和 proc_list 是两个全局链表，要考虑到互斥问题，操作时需将中断关闭，否则会产生错误。 

>创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用**do_fork**函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：
>
> - 调用alloc_proc，首先获得一块用户信息块。
> - 为进程分配一个内核栈。
> - 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
> - 复制原进程上下文到新进程
> - 将新进程添加到进程列表
> - 唤醒新进程
> - 返回新进程号

### 相关宏即函数定义

```c
alloc_proc//proc.c刚完成的 分配一个进程
setup_kstack//proc.c给线程内核栈分配一个大小为KSTACKPAGE(2Page 8KB)的页
copy_mm//proc.c 根据clone_flags对虚拟内存空间进行拷贝 如果和CLONE_VM(pmm.h)一致则共享否则赋值
copy_thread//proc.c 拷贝设置tf以及context(返回eip和中断帧中的栈顶指针esp)中堆栈的信息
hash_proc//proc.c  把进程加到哈希表里
get_pid//proc.c  为新进程创建一个pid
wakup_proc//sched.c  通过将状态置为runable达到唤醒进程的目的
```

```c
static void
copy_thread(struct proc_struct *proc, uintptr_t esp, struct trapframe *tf) {
    //在内核堆栈的顶部设置中断帧大小的一块栈空间
    proc->tf = (struct trapframe *)(proc->kstack + KSTACKSIZE) - 1;
    *(proc->tf) = *tf; //拷贝在kernel_thread函数建立的临时中断帧的初始值
    proc->tf->tf_regs.reg_eax = 0;
    //设置子进程/线程执行完do_fork后的返回值
    proc->tf->tf_esp = esp; //设置中断帧中的栈指针esp
    proc->tf->tf_eflags |= FL_IF; //使能中断
    proc->context.eip = (uintptr_t)forkret;
    proc->context.esp = (uintptr_t)(proc->tf);
}
```
此函数首先在内核堆栈的顶部设置中断帧大小的一块栈空间，并在此空间中拷贝在kernel_thread函数建立的临时中断帧的初始值，并进一步设置中断帧中的栈指针esp和标志寄存器eflags，特别是eflags设置了FL_IF标志，这表示此内核线程在执行过程中，能响应中断，打断当前的执行。执行到这步后，此进程的中断帧就建立好了

代码： 
### `proc.h` 278-346
```c
    int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
     //尝试为进程分配内存
    int ret = -E_NO_FREE_PROC;
    //新进程
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    //内存不足而分配失败
    ret = -E_NO_MEM;
    //LAB4:EXERCISE2 YOUR CODE
    /*
     * Some Useful MACROs, Functions and DEFINEs, you can use them in below implementation.
     * MACROs or Functions:
     *   alloc_proc:   create a proc struct and init fields (lab4:exercise1)
     *   setup_kstack: alloc pages with size KSTACKPAGE as process kernel stack
     *   copy_mm:      process "proc" duplicate OR share process "current"'s mm according clone_flags
     *                 if clone_flags & CLONE_VM, then "share" ; else "duplicate"
     *   copy_thread:  setup the trapframe on the  process's kernel stack top and
     *                 setup the kernel entry point and stack of process
     *   hash_proc:    add proc into proc hash_list
     *   get_pid:      alloc a unique pid for process
     *   wakup_proc:   set proc->state = PROC_RUNNABLE
     * VARIABLES:
     *   proc_list:    the process set's list
     *   nr_process:   the number of process set
     */

    //    1. call alloc_proc to allocate a proc_struct
    //    2. call setup_kstack to allocate a kernel stack for child process
    //    3. call copy_mm to dup OR share mm according clone_flag
    //    4. call copy_thread to setup tf & context in proc_struct
    //    5. insert proc_struct into hash_list && proc_list
    //    6. call wakup_proc to make the new child process RUNNABLE
    //    7. set ret vaule using child proc's pid
    //调用 alloc_proc() 函数申请内存块
    if ((proc = alloc_proc()) == NULL) {
        goto fork_out;
    }
    //将子进程的父节点设置为当前进程
    proc->parent = current;
    //调用 setup_stack() 函数为进程分配一个内核栈
    if (setup_kstack(proc) != 0) {
        //分配内核栈失败
        goto bad_fork_cleanup_proc;
    }
    //调用 copy_mm() 函数复制父进程的内存信息到子进程
    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }
    //调用 copy_thread() 函数复制父进程的中断帧和上下文信息
    copy_thread(proc, stack, tf);

    bool intr_flag;
    //将新进程添加到进程的 hash 列表中
    //全局变量static list_entry_t hash_list[HASH_LIST_SIZE]（所有进程控制块的哈希表）
    //屏蔽中断，intr_flag 置为 1
    local_intr_save(intr_flag);
    {
        //获取当前进程 PID
        proc->pid = get_pid();
        //建立 hash 映射
        hash_proc(proc);
        //将进程加入到进程的链表中
        list_add(&proc_list, &(proc->list_link));
        //进程数加 1
        nr_process ++;
    }
    //恢复中断
    local_intr_restore(intr_flag);
    //唤醒子进程
    wakeup_proc(proc);
    //返回子进程的 pid
    ret = proc->pid;
fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```  
请说明 ucore 是否做到给每个新 fork 的线程一个唯一的 id？请说明你的分析和理由。  
找到 get_pid 函数  

### `proc.h` 137-169  
```c
static int
get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    //两个静态变量last_pid以及next_safe
    //last_pid变量保存上一次分配的PID，而next_safe和last_pid一起表示一段可以使用的PID取值范围(last_pid,next_safe)
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    //同时要求PID的取值范围为[1,MAX_PID]
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    //每次调用 get_pid 时，除了确定一个可以分配的 PID 外，还需要确定 next_safe 来实现均摊以此优化时间复杂度
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        //PID 的确定过程中会检查所有进程的 PID，来确保 PID 是唯一的
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }
    }
    return last_pid;
}
```
可以看到 get_pid 函数有做规避重复的措施，因此只要 get_pid 互斥（例如关闭中断），就可以保证 pid 唯一。  
- 在该函数中使用了两个静态局部变量`next_safe`和`last_pid`，在每次进入`get_pid`函数的时候，这两个变量的数值之间的取值均是合法(尚未使用)的`pid`，如果有严格的`next_safe > last_pid + 1`，就可以直接取`last_pid + 1`作为新的`pid`(`last_pid`就是上一次分配的`PID`)

- 如果`next_safe > last_pid + 1`不成立，则在循环中通过`if (proc->pid == last_pid)`确保不存在任何进程的`pid`与`last_pid`相同，再通过`if (proc->pid > last_pid && next_safe > proc->pid)`保证了不存在任何已经存在的`pid`满足：`last_pid<pid<next_safe`，这样就保证最后能够找到一个满足条件的区间，来获得合法的`pid`