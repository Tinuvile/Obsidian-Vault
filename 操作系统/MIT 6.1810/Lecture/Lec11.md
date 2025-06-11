# Thread switching

---

## 线程概述

一个线程可以理解成一个串行执行代码的单元。线程具有状态，它包含程序计数器、保存变量的寄存器和程序栈。操作系统中线程系统的工作就是管理多个线程的运行。多线程的并行运行主要有两个策略：
- 一是在多核处理器上使用多个CPU，每个CPU都可以运行一个线程。
- 二是一个CPU在多个线程间来回切换。

不同线程系统之间最主要的区别是：线程之间是否会共享内存。`xv6`共享了内存，并且支持内核线程的概念，对于每个用户进程都有一个内核线程来执行来自用户进程的系统调用。所有的内核线程共享了内核内存。

同时，`xv6`也有另一种线程，每一个用户进程都有独立的内存地址空间，并且包含了一个线程，这个线程控制了用户进程代码指令的执行，它们之间没有共享内存。

`Linux`中还允许一个用户进程中包含多个线程，进程中的多个线程共享进程的地址空间。

---

## `xv6`线程调度

实现内核中的线程系统存在以下挑战：
- 一是如何实现线程间的切换。停止一个线程的运行并启动另一个线程的过程通常被称为线程调度*Scheduling*。`xv6`为每个CPU核都创建了一个线程调度器*Scheduler*。
- 二是在切换时，需要保存并恢复线程的状态，需要确定保存哪些信息，以及在哪里保存。
- 三是如何处理运算密集型线程*compute bound thread*，这种线程往往无法自主让出CPU，需要外部使它撤回对CPU的控制。

首先是如何处理运算密集型线程，方法是利用定时器中断。每个CPU核上，都存在一个硬件设备，它会定时产生中断。与其他操作系统一样，`xv6`会将这个中断传输到内核中，由于中断处理程序的优先级更高，它可以在某个时间触发并将程序的控制权从用户空间代码转换到内核中的中断处理程序。而位于内核的定时器中断处理程序，会自愿将CPU让出给线程调度器（`yield`）。

这样的处理流程叫*pre-emptive scheduling*，*pre-emptive*意为即使用户代码本身没有让出CPU，定时器中断仍然会将CPU控制器拿走，并让出线程调度器。与之相反的是*voluntary scheduling*。

在`xv6`中线程调度实现是：
- 定时器中断会强制将CPU控制权从用户进程给到内核，这是*pre-emptive scheduling*。
- 内核中用户进程对应的内核进程会代表用户进程让出CPU，这是*voluntary scheduling*。

在执行时，操作系统需要区分几类线程。当前在CPU上运行的线程、一旦CPU有空闲就想运行的线程、不想运行的线程（可能在等待I/O或者其他事件）。

---

## `xv6`线程切换

我们可能会运行多个用户空间进程，它们都各自包含一个用户程序栈，并且当进程运行时，它在RISC-V处理器中会有程序计数器和寄存器。当用户程序运行时，实际上是用户进程中的一个用户线程在运行。如果程序执行了一个系统调用或者因为响应中断走到了内核中，那么相应的用户空间状态会被保存到程序的[[操作系统/MIT 6.1810/Book/Chapter4#^004747|trapframe]]，同时属于这个用户程序的内核线程被激活。处理完成后如需返回用户空间，`trapframe`中保存的用户进程状态会被恢复。

除了系统调用，用户进程也可能因为CPU需要响应定时器中断或其他而进入内核空间，如*pre-emptive scheduling*，会通过定时器中断将CPU运行切换到另一个用户进程。在定时器中断程序中，如果`xv6`决定从一个用户进程切换到另一个用户进程，那首先在内核中的第一个进程的内核线程会被切换到第二个进程的内核线程，之后再在第二个进程的内核线程中返回到用户空间的第二个进程，这里的返回也是通过`trapframe`中保存的用户进程状态完成。

大概进程切换的过程是：
- 先从第一个用户进程进入内核，保存用户进程状态并运行第一个用户进程的内核线程。
- 再从第一个用户进程的内核线程切换到第二个用户进程的内核线程。
- 然后第二个用户进程的内核线程暂停自己，恢复第二个用户进程的用户寄存器。
- 最后返回第二个用户进程继续执行。

现在假设有进程$P1$正在运行，进程$P2$是`RUNNABLE`，当前不在运行。然后在`xv6`中有两个CPU核，即$CPU0$和$CPU1$。

一个更完整的过程是：
- 首先，一个定时器中断强迫CPU从用户空间进程切换到内核，`trampoline`代码将用户寄存器保存于用户进程对应的`trapframe`对象中
- 之后在内核中运行`usertrap`，来实际执行相应的中断处理程序。这时，CPU正在进程$P1$的内核线程和内核栈上，执行内核中普通的C代码
- 假设进程$P1$对应的内核线程决定它想出让CPU，它会做很多工作，但是最后它会调用`swtch`函数，这是整个线程切换的核心函数之一；
- `swtch`函数会保存用户进程$P1$对应内核线程的寄存器至`context`对象。所以目前为止有两类寄存器：用户寄存器存在`trapframe`中，内核线程的寄存器存在`context`中

但实际上`swtch`函数并不是直接从一个内核线程切换到另一个内核线程。`xv6`中，一个CPU上运行的内核线程可以直接切换到的是这个CPU对应的调度器线程。所以如果我们运行在$CPU0$，`swtch`函数会恢复之前为$CPU0$的调度器线程保存的寄存器和栈指针，之后就在调度器线程的`context`下执行`schedulder`函数。

`scheduler`函数会做一些清理工作，然后通过进程表单找到下一个`RUNNABLE`进程，假设找到$P2$，然后再次调用`swtch`函数：
- 先保存自己的寄存器到调度器线程的`context`对象
- 然后找到进程$P2$之前保存的`context`，恢复其中的寄存器
- 因为进程$P2$在进入`RUNNABLE`状态之前，如前面进程$P1$一样，必然也调用了`swtch`函数。所以之前的`swtch`函数会被恢复，并返回到进程$P2$所在的系统调用或者中断处理程序中，因为$P2$进程之前调用`swtch`函数必然在系统调用或者中断处理程序中
- 当内核程序执行完成之后，`trapframe`中的用户寄存器会被恢复，最后用户进程$P2$就恢复运行了

每个CPU都有一个完全不同的调度器线程，它也是一种内核线程，有自己的`context`对象。任何运行在$CPU1$上的进程，当它决定出让CPU，它都会切换到$CPU1$对应的调度器线程，并由调度器线程切换到下一个进程。

> [!note] `context`保存位置
> - 每一个用户进程有一个对应的内核线程，它的`context`对象保存在用户进程对应的`proc`结构体中
> - 每一个调度器线程也有自己的`context`对象，它保存在`cpu`结构体中

---

## 示例程序与代码

先看`proc.h`中的`proc`结构体。

```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

可以看到保存用户空间线程寄存器的`trapframe`字段，以及保存内核线程寄存器的`context`字段，保存当前进程的内核栈的`kstack`字段（这是进程在内核中执行时保存函数调用的位置），`state`字段保存当前进程状态（`RUNNING`、`RUNNABLE`或是`SLEEPING`），`lock`字段保护很多数据，比如`state`字段。

> [!tip] 如何区分不同进程的内核线程
> - 每个进程都有不同的内核栈，`proc`结构体中的`kstack`字段指向它
> - 任何内核代码都可以通过调用`myproc`来获取当前CPU正在运行的进程。内核线程可以通过调用这个函数知道自己属于哪个用户进程
> 
> `myproc`函数使用`tp`寄存器来获取当前CPU核的ID，并使用这个ID在一个保存了所有CPU上运行的进程的结构体数组中找到对应的`proc`结构体  

定时器中断会将CPU切换到另一个进程中。在下面的函数中，`else if(scause == 0x8000000000000005L)`会识别并响应定时器中断。

```c
// check if it's an external interrupt or software interrupt,
// and handle it.
// returns 2 if timer interrupt,
// 1 if other device,
// 0 if not recognized.
int
devintr()
{
  uint64 scause = r_scause();

  if(scause == 0x8000000000000009L){
    // this is a supervisor external interrupt, via PLIC.

    // irq indicates which device interrupted.
    int irq = plic_claim();

    if(irq == UART0_IRQ){
      uartintr();
    } else if(irq == VIRTIO0_IRQ){
      virtio_disk_intr();
    } else if(irq){
      printf("unexpected interrupt irq=%d\n", irq);
    }

    // the PLIC allows each device to raise at most one
    // interrupt at a time; tell the PLIC the device is
    // now allowed to interrupt again.
    if(irq)
      plic_complete(irq);

    return 1;
  } else if(scause == 0x8000000000000005L){
    // timer interrupt.
    clockintr();
    return 2;
  } else {
    return 0;
  }
}
```

在中断的位置，`usertrap`函数会通过`devintr`函数来处理，`devintr`函数返回`2`到`usertrap`函数中，然后运行到`yield`函数。

```c
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause 0x%lx pid=%d\n", r_scause(), p->pid);
    printf("            sepc=0x%lx stval=0x%lx\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

`yield`函数是整个线程切换的第一步。

```c
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

它首先获取进程的锁，然后修改进程的一些状态，进入`sched`函数。

```c
// Switch to scheduler.  Must hold only p->lock
// and have changed proc->state. Saves and restores
// intena because intena is a property of this
// kernel thread, not this CPU. It should
// be proc->intena and proc->noff, but that would
// break in the few places where a lock is held but
// there's no process.
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```

`sched`函数只做了一些合理性检查，发现异常就`panic`，然后调用`swtch`函数。

`swtch`函数将当前的内核线程的寄存器保存到`p->context`中，另一个参数`&mycpu->context`，`mycpu`表示当前CPU的结构体，结构体中的`context`保存了当前CPU核的调度器线程的寄存器。所以`swtch`函数保存完当前内核线程的内核寄存器后，就会恢复当前CPU核的调度器线程的寄存器，并继续执行当前CPU核的调度器线程。

```asm
.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret
```

首先，`ra`寄存器被保存到`a0`寄存器指向的地址。`a0`寄存器对应`swtch`函数的第一个参数，当前线程的`context`对象地址；`a1`寄存器对应第二个参数，即将切换到的调度器线程的`context`对象地址。

函数的上半部分将当前的寄存器保存到当前线程对应的`context`对象中，下半部分将调度器线程的寄存器恢复到CPU的寄存器中，然后函数返回。`ra`寄存器指向的是它返回的地址，也就是`scheduler`函数。

> [!faq] 为什么RISC-V中32个寄存器，`swtch`函数只保存并恢复了14个
> `swtch`是从C代码调用的，`Caller Saved Register`会被C编译器保存到当前的栈上，大概有15-18个，因此在`swtch`中只需要处理C编译器不会报错但是有用的。

然后再看一下`sp`寄存器（*Stack Pointer*），它是当前进程的内核栈地址，由虚拟内存系统映射在了一个高地址。

最后，通过执行`ret`指令，就可以返回调度器线程中。

> [!faq] 为什么`swtch`要用汇编实现
> C语言中无法与寄存器交互，在C中嵌套汇编语言与直接定义一个汇编函数是一样的，它的操作在C语言的层级之下，所以不能使用C语言。

```c
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for(;;){
    // The most recent process to run may have had interrupts
    // turned off; enable them to avoid a deadlock if all
    // processes are waiting.
    intr_on();

    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
        found = 1;
      }
      release(&p->lock);
    }
    if(found == 0) {
      // nothing to run; stop running on this core until an interrupt.
      intr_on();
      asm volatile("wfi");
    }
  }
}
```

这里我们运行在CPU拥有的调度器线程中，并且正好在之前调用`swtch`函数的返回状态，现在就可以释放锁了，关于锁的详细问题会在下一章。

> [!important]
> 当调用`swtch`函数时，实际上是一个线程对`swtch`的调用切换到了另一个线程对`swtch`的调用。



