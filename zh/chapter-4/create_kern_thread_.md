# 【实现】创建并执行内核线程

既然建立了进程控制块，我们就可以通过进程控制块来创建具体的进程了。首先，我们考虑最简单的内核线程，它通常只是内核中的一小段代码或者函数，没有用户空间。而由于在操作系统启动后，已经对整个核心内存空间进行了管理，通过设置页表建立了核心虚拟空间（即boot_cr3指向的二级页表描述的空间）。所以内核中的所有线程都不需要再建立各自的页表，只需共享这核心虚拟空间就可以访问整个物理内存了。

## 创建第0个内核线程idleproc

在init.c::kern_init函数调用了proc.c::proc_init函数。proc_init函数启动了创建内核线程的步骤。	首先当前的执行上下文（从kern_init 启动至今）就可以看成是一个内核线程的上下文，为此 ucore 通过给当前执行的上下文分配一个进程控制块以及对它进行相应初始化而将其打造成第0个内核线程 -- idleproc。具体步骤如下：

首先调用alloc_proc函数来通过kmalloc函数获得proc_struct结构的一块内存—proc，这就是第0个进程控制块了，并把proc进行初步初始化（即把proc_struct中的各个域清零）。但有些域设置了特殊的值：

     proc->state = PROC_UNINIT; //设置进程为“初始”态
     proc->pid = -1;              //进程的pid还没设置好
     proc->cr3 = boot_cr3;       //进程在内核中使用的内核页表的起始地址

上述三条语句中,第一条设置了进程的状态为“初始”态，这表示进程已经 “出生”了，正在获取资源茁壮成长中；第二条语句设置了进程的pid为-1，这表示进程的“身份证号”还没有办好；第三条语句表明进程如果在内核运行，则采用为内核建立的页表，即设置了在内核的页表的起始地址，这也可进一步看出所有进程的内核虚地址空间（也包括物理地址空间）是相同的。既然内核进程共用一个映射内核空间的页表，这表示所有这些内核空间对所有内核进程都是“可见”的，所以更精确地说，这些内核进程都应该是内核线程。

接下来，proc_init函数对idleproc内核线程进行进一步初始化：

    idleproc->pid = 0;
    idleproc->state = PROC_RUNNABLE;
    idleproc->kstack = (uintptr_t)bootstack;
    idleproc->need_resched = 1;
    set_proc_name(idleproc, "idle");

需要注意前4条语句。第一条语句给了idleproc合法的身份证号--0，这名正言顺地表明了idleproc是第0个内核线程。“0”是第一个的表示方法是计算机领域所特有的，比如C语言定义的第一个数组元素的小标也是“0”。第二条语句改变了idleproc的状态，使得它从“出生”转到了“准备工作”，就差ucore调度它执行了。第三条语句设置了idleproc所使用的内核栈的起始地址。需要注意以后的其他进程/线程的内核栈都需要通过分配获得，因为ucore启动时设置的内核栈直接分配给idleproc使用了。第四条很重要，因为ucore希望当前CPU应该做更有用的工作，而不是运行idleproc这个“无所事事”的内核线程，所以把idleproc->need_resched设置为“1”，结合idleproc的执行主体--cpu_idle函数的实现，可以清楚看出如果当前idleproc在执行，则只要此标志为1，马上就调用schedule函数要求调度器切换其他进程执行。

**【问题】为何说idleproc的执行主体是cpu_idle函数？**

## 创建第1个内核线程initproc

第0个内核线程主要工作是完成内核中各个子系统的初始化，然后就通过执行cpu_idle函数开始过退休生活了。但接下来还需创建其他进程来完成各种工作，但idleproc自己不想做，于是就通过调用kernel_thread函数创建了一个内核线程init_main。在proj10中，这个子内核线程的工作就是输出一些字符串，然后就返回了（参看init_main函数）。但在后续的proj中，init_main的工作就是创建特定的其他内核线程或用户进程。下面我们来分析一下创建内核线程的函数kernel_thread：

    kernel_thread(int (*fn)(void *), void *arg, uint32_t clone_flags) {
        struct trapframe tf_struct;
        memset(&tf_struct, 0, sizeof(struct trapframe));
        tf_struct.tf_cs = KERNEL_CS;
        tf_struct.tf_ds = tf_struct.tf_es = tf_struct.tf_ss = KERNEL_DS;
        tf_struct.tf_regs.reg_ebx = (uint32_t)fn;
        tf_struct.tf_regs.reg_edx = (uint32_t)arg;
        tf_struct.tf_eip = (uint32_t)kernel_thread_entry;
        return do_fork(clone_flags | CLONE_VM, 0, &tf_struct);
    }

注意，kernel_thread函数采用了局部变量tf来放置保存内核线程的临时中断帧，并把中断帧的指针传递给do_fork函数，而do_fork函数会调用copy_thread函数来在新创建的进程内核栈上专门给进程的中断帧分配一块空间。

给中断帧分配完空间后，就需要构造新进程的中断帧，具体过程是：首先给tf进行清零初始化，并设置中断帧的代码段（tf.tf_cs）和数据段(tf.tf_ds/tf_es/tf_ss)为内核空间的段（KERNEL_CS/ KERNEL_DS），这实际上也说明了initproc内核线程在内核空间中执行。而initproc内核线程从哪里开始执行呢？tf.tf_eip的指出了是kernel_thread_entry（位于kern/process/entry.S中），kernel_thread_entry是entry.S中实现的汇编函数，它做的事情很简单：

    kernel_thread_entry:        # void kernel_thread(void)

        pushl %edx              # push arg
        call *%ebx              # call fn

        pushl %eax              # save the return value of fn(arg)
        call do_exit            # call do_exit to terminate current thread

从上可以看出，kernel_thread_entry函数主要为内核线程的主体fn函数做了一个准备开始和结束运行的“壳”，及把函数fn的参数arg（保存在edx寄存器中）压栈，然后调用fn函数，把函数返回值eax寄存器内容压栈，调用do_exit函数退出线程执行。

do_fork是创建线程的主要函数。kernel_thread函数通过调用do_fork函数最终完成了内核线程的创建工作。下面我们来分析一下do_fork函数的实现。do_fork函数主要做了以下6件事情：

1．分配并初始化进程控制块（alloc_proc函数）；

2．分配并初始化内核栈（setup_stack函数）；

3．根据clone_flag标志复制或共享进程内存管理结构（copy_mm换生）；

4．设置进程在内核（将来也包括用户态）正常运行和调度所需的中断帧和执行上下文（copy_thread函数）；

5．把设置好的进程控制块放入hash_list和proc_list两个全局进程链表中；

6．自此，进程已经准备好执行了，把进程状态设置为“就绪”态。

这里需要注意的是，如果上述前3步执行没有成功，则需要做对应的出错处理，把相关已经占有的内存释放掉。copy_mm函数目前只是把current->mm设置为NULL，这是由于目前proj10只能创建内核线程，proc->mm描述的是进程用户态空间的情况，所以目前mm还用不上。copy_thread函数做的事情比较多，代码如下：

    static void
    copy_thread(struct proc_struct *proc, uintptr_t esp, struct trapframe *tf) {
        proc->tf = (struct trapframe *)(proc->kstack + KSTACKSIZE) - 1;
        *(proc->tf) = *tf;
        proc->tf->tf_regs.reg_eax = 0;
        proc->tf->tf_esp = esp;
        proc->tf->tf_eflags |= FL_IF;

        proc->context.eip = (uintptr_t)forkret;
        proc->context.esp = (uintptr_t)(proc->tf);
    }

此函数首先在内核堆栈的顶部设置中断帧大小的一块栈空间，并在此空间中拷贝在kernel_thread函数建立的临时中断帧的初始值，并进一步设置中断帧中的栈指针esp和标志寄存器eflags，特别是eflags设置了FL_IF标志，这表示此内核线程在执行过程中，能响应中断，打断当前的执行。执行到这步后，此进程的中断帧就建立好了，对于initproc而言，它的中断帧如下所示：

    //所在地址位置
    initproc->tf= (proc->kstack+KSTACKSIZE) – sizeof (struct trapframe);   
    //具体内容
    initproc->tf.tf_cs = KERNEL_CS;
    initproc->tf.tf_ds = initproc->tf.tf_es = initproc->tf.tf_ss = KERNEL_DS;
    initproc->tf.tf_regs.reg_ebx = (uint32_t)init_main;
    initproc->tf.tf_regs.reg_edx = (uint32_t) ADDRESS of "Hello world!!";
    initproc->tf.tf_eip = (uint32_t)kernel_thread_entry;
    initproc->tf.tf_regs.reg_eax = 0; 
    initproc->tf.tf_esp = esp;
    initproc->tf.tf_eflags |= FL_IF;


设置好中断帧后，最后就是设置initproc的执行现场（也称进程上下文, process context）了。只有设置好执行现场后，一旦ucore调度器选择了initproc执行，就需要根据initproc->context中保存的执行现场来恢复initproc的执行这里设置了initproc的执行现场中主要的两个信息：上次停止执行时的下一条指令地址context.eip和上次停止执行时的堆栈地址context.esp。其实initproc还没有执行过，所以这其实就是initproc实际执行的第一条指令地址和堆栈指针。可以看出，由于initproc的中断帧占用了实际给initproc分配的栈空间的顶部，所以initproc就只能把栈顶指针context.esp设置在initproc的中断帧的起始位置。根据context.eip的赋值，可以知道initproc实际开始执行的地方在forkret函数处。至此，initproc内核线程已经做好准备执行了。

## 调度并执行内核线程initproc 

在ucore执行完proc_init函数后，就创建好了两个内核线程：idleproc和initproc，这时ucore当前的执行现场就是idleproc，等到执行到init函数的最后一个函数cpu_idle之前，ucore的所有初始化工作就结束了，idleproc将通过执行cpu_idle函数让出CPU，给其它内核线程执行，具体过程如下：

    void
    cpu_idle(void) {
        while (1) {
            if (current->need_resched) {
                schedule();
         ……

首先，判断当前内核线程idleproc的need_resched是否不为0，回顾本节中“创建第一个内核线程idleproc”中的描述，proc_init函数在初始化idleproc中，就把idleproc->need_resched置为1了，所以会马上调用schedule函数找其他处于“就绪”态的进程执行。

ucore在proc10中只实现了一个最简单的FIFO调度器，其核心就是schedule函数。它的执行逻辑很简单：

1．设置当前内核线程current->need_resched为0；

2．在proc_list队列中查找下一个处于“就绪”态的线程或进程next；

3．找到这样的进程后，就调用proc_run函数，保存当前进程current的执行现场（进程上下文），恢复新进程的执行现场，完成进程切换。

至此，新的进程next就开始执行了。由于在proc10中只有两个内核线程，且idleproc要让出CPU给initproc执行，我们可以看到schedule函数通过查找proc_list进程队列，只能找到一个处于“就绪”态的initproc内核线程。并通过proc_run和进一步的switch_to函数完成两个执行现场的切换，具体流程如下：

1．让current指向next内核线程initproc；

2．设置任务状态段ts的特权态0下的栈顶指针esp0为next内核线程initproc的内核栈的栈顶，即next->kstack + KSTACKSIZE ；

3．设置CR3寄存器的值为next内核线程initproc的页目录表起始地址next->cr3，这实际上是完成进程间的页表切换；

4．由switch_to函数完成具体的两个线程的执行现场切换，即切换各个寄存器，当switch_to函数执行完“ret”指令后，就切换到initproc执行了。

这里需要注意，在第二步设置任务状态段ts的特权态0下的栈顶指针esp0的目的是建立好内核线程或将来用户线程在执行特权态切换（从特权态0<-->特权态3，或从特权态3<-->特权态3）时能够正确定位处于特权态0时进程的内核栈的栈顶，而这个栈顶其实放了一个trapframe结构的内存空间。如果是在特权态3发生了中断/异常/系统调用，则CPU会从特权态3-->特权态0，且CPU从此栈顶开始压栈来保存被中断/异常/系统调用打断的用户态执行现场；如果是在特权态0发生了中断/异常/系统调用，则CPU会从特权态还是0，且 CPU从当前栈指针esp所指的位置开始压栈保存被中断/异常/系统调用打断的内核态执行现场。反之，当执行完对中断/异常/系统调用打断的处理后，最后会执行一个“iret”指令。在执行此指令之前，CPU的当前栈指针esp一定指向上次产生中断/异常/系统调用时CPU保存的被打断的指令地址CS和EIP，“iret”指令会根据ESP所指的保存的址CS和EIP恢复到上次被打断的地方继续执行。

在页表设置方面，由于idleproc和initproc都是共用一个内核页表boot_cr3，所以此时第三步其实没用，但考虑到以后的进程有各自的页表，其起始地址各不相同，只有完成页表切换，才能确保新的进程能够正常执行。

第四步proc_run函数调用switch_to函数，参数是前一个进程和后一个进程的执行现场context。在上一节“设计进程控制块”中，描述了context结构包含的要保存和恢复的寄存器。我们在看看switch.S中的switch_to函数的执行流程：

    .globl switch_to
    switch_to:                      # switch_to(from, to)

        # save from's registers
        movl 4(%esp), %eax          # eax points to from
        popl 0(%eax)                # save eip !popl
        movl %esp, 4(%eax)
        ……
        movl %ebp, 28(%eax)
        # restore to's registers
        movl 4(%esp), %eax          # not 8(%esp): popped return address already
                                    # eax now points to to
        movl 28(%eax), %ebp
        ……
        movl 4(%eax), %esp
        pushl 0(%eax)               # push eip
        ret
      
首先，保存前一个进程的执行现场，前两条汇编指令（如下所示）保存了进程在返回switch_to函数后的指令地址到context.eip中

	movl 4(%esp), %eax          # eax points to from
    popl 0(%eax)                # save eip !popl
  
在接下来的7条汇编指令完成了保存前一个进程的其他7个寄存器到context中的相应域中。至此前一个进程的执行现场保存完毕。接下来就是恢复向一个进程的执行现场，这其实就是上述保存过程的逆执行过程，即从context的高地址的域ebp开始，逐一把相关域的值赋值给对应的寄存器，倒数第二条汇编指令“pushl 0(%eax)”其实把context中保存的下一个进程要执行的指令地址context.eip放到了堆栈顶，这样接下来执行最后一条指令“ret”时，会把栈顶的内容赋值给EIP寄存器，这样就切换到下一个进程执行了，即当前进程已经是下一个进程了。

再回到proj10中，ucore会执行进程切换，让initproc执行。在对initproc进行初始化时，设置了initproc->context.eip = (uintptr_t)forkret，这样，当执行switch_to函数并返回后，initproc将执行其实际上的执行入口地址forkret。而forkret会调用位于kern/trap/trapentry.S中的forkrets函数执行，具体代码如下：

    .globl __trapret
    __trapret:
        # restore registers from stack
        popal

        # restore %ds and %es
        popl %es
        popl %ds

        # get rid of the trap number and error code
        addl $0x8, %esp
        iret

    .globl forkrets
    forkrets:
        # set stack to this new process's trapframe
        movl 4(%esp), %esp       //把esp指向当前进程的中断帧
        jmp __trapret

可以看出，forkrets函数首先把esp指向当前进程的中断帧，从_trapret开始执行到iret前，esp指向了current->tf.tf_eip，而如果此时执行的是initproc，则current->tf.tf_eip= kernel_thread_entry，initproc->tf.tf_cs = KERNEL_CS，所以当执行完iret后，就开始在内核中执行kernel_thread_entry函数了，而initproc->tf.tf_regs.reg_ebx = init_main，所以在kernl_thread_entry中执行“call %ebx”后，就开始执行initproc的主体了。Initprocde的主体函数很简单就是输出一段字符串，然后就返回到kernel_tread_entry函数，并进一步调用do_exit执行退出操作了。本来do_exit应该完成一些资源回收工作等，但这些不是proj10涉及的，而是由后续的proj来完成。
    至此，proj10的主要工作描述完毕