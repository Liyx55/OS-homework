# lab6
## 【实验题目】实验6：调度器
## 【实验目的】
    •	理解操作系统的调度管理机制
    •	熟悉 ucore 的系统调度器框架，以及缺省的Round-Robin 调度算法
    •	基于调度器框架实现一个(Stride Scheduling)调度算法来替换缺省的调度算法  

## 【实验要求】
    •	为了实现实验的目标，实验提供了3个基本练习和2个扩展练习，要求完成实验报告。
    •	练习0：填写已有实验
    •	练习1：使用 Round Robin 调度算法（不需要编码）
    •	练习2：实现 Stride Scheduling 调度算法（需要编码）
    •	练习3：阅读分析源代码，结合中断处理和调度程序，再次理解进程控制块中的trapframe和context在进程切换时作用（不需要编码）  

## 【实验方案】
### **练习0**：  
合并代码使用meld工具
![](picture/lab6-1.png)
部分函数需要被修改  
**alloc_proc**:  
被添加的部分为  
```c
    list_init(&(proc->run_link));
    proc->time_slice = 0;
    memset(&(proc->lab6_run_pool), 0, sizeof(proc->lab6_run_pool));
    proc->lab6_stride = 0;
    proc->lab6_priority = 0;
```
新加的代码用于初始化运行队列，时间片，运行池的节点，已走的步长和优先级。  
**trap_dispatch**:  
被修改的部分为
```c
        //print_ticks();
        assert(current!=NULL);
        //current->need_resched = 1;
        run_timer_list();
```
每次时钟中断都执行一次run_timer_list。  
### **练习1**：    
（不需要编程）  

    •	完成练习0后，建议大家比较一下（可用kdiff3等文件比较软件）个人完成的lab5和练习0完成后的刚修改的lab6之间的区别，分析了解lab6采用RR调度算法后的执行过程。执行make grade，大部分测试用例应该通过。但执行priority.c应该过不去。
    •	请在实验报告完成下面要求：
    •	请理解并分析sched_calss中各个函数指针的用法，并接合Round Robin 调度算法描述ucore的调度执行过程
    •	请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计  

（1）在sched.h中我们可以找到sched_class的定义
```c
// The introduction of scheduling classes is borrrowed from Linux, and makes the 
// core scheduler quite extensible. These classes (the scheduler modules) encapsulate 
// the scheduling policies. 
//调度类是从Linux中引入的，这使得核心调度程序具有相当的可扩展性。这些类(调度程序模块)封装了调度策略。
struct sched_class {
    // 调度器名称
    const char *name;
    // 初始化运行队列
    void (*init)(struct run_queue *rq);
    // 将进程放入队列中，必须使用rq_lock调用
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    // 将进程从队列中移除，必须使用rq_lock调用
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    // 选择下一个可运行的任务
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    // 更新调度器的时钟信息
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
    /* for SMP support in the future
     *  load_balance
     *     void (*load_balance)(struct rq* rq);
     *  get some proc from this rq, used in load_balance,
     *  return value is the num of gotten proc
     *  int (*get_proc)(struct rq* rq, struct proc* procs_moved[]);
     */
};
```  
RR调度算法:
    
    RR调度算法的调度思想是让所有runnable态的进程分时轮流使用CPU时间。RR调度器维护当前runnable进程的有序运行队列。当前进程的时间片用完之后，调度器将当前进程放置到运行队列的尾部，再从其头部取出进程进行调度。RR调度算法的就绪队列在组织结构上也是一个双向链表，只是增加了一个成员变量，表明在此就绪进程队列中的最大执行时间片。而且在进程控制块proc_struct中增加了一个成员变量time_slice，用来记录进程当前的可运行时由于间片段。这是RR调度算法需要考虑执行进程的运行时间不能太长。在每个timer到时的时候，操作系统会递减当前执行进程的time_slice，当time_slice为0时，就意味着这个进程运行了一段时间（这个时间片段称为进程的时间片），需要把CPU让给其他进程执行，于是操作系统就需要让此进程重新回到rq的队列尾，且重置此进程的时间片为就绪队列的成员变量最大时间片max_time_slice值，然后再从rq的队列头取出一个新的进程执行。

其中包含了调度方法的名字、运行队列指针与四个函数指针，四个函数指针作为接口从而可以实现不同的调度方法（例如RR与Stride），四个接口的用途总结如下：  

	enqueue: 入队，将进程放入调度队列中。
	dequeue: 出队，将进程从调度队列中移除
	pick_next: 选取，择选下一个将被运行的进程。
	proc_tick: 触发，检测是否需要调度  

接下来结合RR算法来分析Ucore的调度过程  
**RR_init**:
```c
static void
RR_init(struct run_queue *rq) {
    list_init(&(rq->run_list));
    rq->proc_num = 0;
}
```
用于初始化运行队列，调用list_init来初始化，同时将队列进程数设为0  
**RR_enqueue**:
```c
static void
RR_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(list_empty(&(proc->run_link)));
    list_add_before(&(rq->run_list), &(proc->run_link));
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num ++;
}
```
RR入队函数，将新加进来的队列放到运行队列的末尾，符合FIFO，将进程的队列指针指向当前队列，最后使队列数自增。  
**RR_dequeue**:
```c
static void
RR_dequeue(struct run_queue *rq, struct proc_struct *proc) {
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
    list_del_init(&(proc->run_link));
    rq->proc_num --;
}
```
RR出队函数，把运行队列的第一个进程去除，并且进程数减一。  
**RR_pick_next**:  
```c
static struct proc_struct *
RR_pick_next(struct run_queue *rq) {
    list_entry_t *le = list_next(&(rq->run_list));
    if (le != &(rq->run_list)) {
        return le2proc(le, run_link);
    }
    return NULL;
}
```
RR选取进程函数，根据FIFO，选取rq的第一个就绪进程作为下一个进程。通过list_next函数的调用，会从队头选择一个进程，代表当前应该去执行的那个进程，交给schedule调用。  
**schedule**:  
```c
void
schedule(void) {
    bool intr_flag;
    struct proc_struct *next;
    local_intr_save(intr_flag);
    {
        current->need_resched = 0;
        if (current->state == PROC_RUNNABLE) {
            sched_class_enqueue(current);
        }
        if ((next = sched_class_pick_next()) != NULL) {
            sched_class_dequeue(next);
        }
        if (next == NULL) {
            next = idleproc;
        }
        next->runs ++;
        if (next != current) {
            proc_run(next);
        }
    }
    local_intr_restore(intr_flag);
}

```
从这里可以看出来，如果选不出来有处在runnable的进程，那么返回NULL，并将执行权交给内核线程idle，idle会不断调用schedule，直到整个系统出现下一个可以执行的进程。
**RR_proc_tick**:  
```c
static void
RR_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
    if (proc->time_slice > 0) {
        proc->time_slice --;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
}
```
RR触发函数，产生时钟中断的时候，会调用tick函数，每次时间片到了的时候会产生时钟中断，time_slice-1，如果time_slice降为零，即为时间片用完。一旦时间片用完了，那么就需要把该进程PCB中的need_resched置为1，然后由schedule函数进行调度。首先将当前进程的时间片减一，若其已没有（或用完）可用的时间片，则说明需要调度，将need_resched设为1  
**执行过程**.  
通过这几个函数，我们可以知道在时间轮转法(RR)调度下的进程调度过程。RR算法使用一个双向链表作为运行队列，遵循FIFO的规律，后到的进程会被放在队列尾。为了实现时间轮转的功能，为proc_struct添加了一个time_slice的成员变量，其表示进程还剩余的时间片长度，每次使用proc_tick(在run_timer_list被调用)时，time_slice就会减一。如果time_slice减为0，则说明当前进程时间片用完了，需要进行调度，此时如果该进程没有执行完，就会被放到队列尾，在从队列头取一个进程执行，以达到时间轮转的目的。

（2）  

	以RR算法为蓝本进行改进实现多级反馈队列调度算法：
	a. 设置多个不同优先级的运行队列，优先调度高优先级队列中的进程。
	b. 同一个优先级队列中的进程使用RR算法进行调度。
	c. 为防止发生饥饿，每隔一定周期需将一定低优先级的进程的优先级上升
	d. 同理，当高优先级队列的一个进程占用CPU过长时，将其优先级下降

具体实现：

假设进程一共有4个调度优先级，分别为0、1、2、3，其中0位最高优先级，3位最低优先级。为了支持4个不同的优先级，在运行队列中开4个队列，分别命名为rq -> run_list[0..3]。除此之外，在proc_struct中加入priority成员表示该进程现在所处的优先级，初始化为0。

**MLFQ_enqueue**:
```c
static void
MLFQ_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(list_empty(&(proc->run_link)));
    if (proc -> time_slice == 0 && proc -> priority != 3) {
        ++(proc -> priority);
    }
    list_add_before(&(rq->run_list[proc ->priority]), &(proc->run_link));
    proc->time_slice = (rq->max_time_slice << proc -> priority);
    proc->rq = rq;
    rq->proc_num ++;
}
```
RR入队函数，将新加进来的队列放到运行队列的末尾，符合FIFO，将进程的队列指针指向当前队列，最后使队列数自增。  
**MLFQ_dequeue**:
```c
static void
MLFQ_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
    list_del_init(&(proc->run_link));
    rq->proc_num --;
}
```
将proc从相应优先级的运行队列中删去。 
**MLFQ_pick_next**:  
```c
static struct proc_struct *
MLFQ_pick_next(struct run_queue *rq) {
    int p = rand() % (8 + 4 + 2 + 1);
    int priority;
    if (p >= 0 && p < 8) {
        priority = 0;
    } else if (p >= 8 && p < 12) {
        priority = 1;
    } else if (p >= 12 && p < 14) {
        priority = 2;
    } else priority = 3;
    list_entry_t *le = list_next(&(rq->run_list[priority]));
    if (le != &(rq->run_list[priority])) {
        return le2proc(le, run_link);
    } else {
        for (int i = 0; i < 3; ++i) {
            le = list_next(&(rq->run_list[i]));
            if (le != &(rq -> run_list[i])) return le2proc(le, run_link);
        }
    }
    return NULL;
}
```
为了避免优先级较低的进程出现饥饿，对每个优先级设置一定的选中概率，高优先级是低优先级选中率的两倍，选出一个优先级后，返回相应优先级队列的第一个进程。
**RR_proc_tick**: 和RR算法的RR_proc_tick类似。
```c
static void
RR_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
    if (proc->time_slice > 0) {
        proc->time_slice --;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
}
```