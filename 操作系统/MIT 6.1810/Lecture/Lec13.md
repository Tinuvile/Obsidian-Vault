# Sleep & Wake up

---

## 线程切换过程中锁的限制

在`xv6`中，调用`swtch`函数会从一个线程切换到另一个线程，通常是在用户进程的内核线程和调度器线程之间切换。调用前，先获取线程对应的用户进程的锁，然后调用`swtch`函数切换到调度器线程，调度器线程再释放进程锁。
- 一个进程想进入休眠状态，它会先获取自己的锁
- 然后将自己的状态从`RUNNING`设置为`RUNNABLE`
- 再调用`swtch`函数，实际是调用`sched`函数，再在`sched`函数中调用`swtch`函数
- `swtch`函数将当前的线程切换到调度器线程
- 调度器线程之前也调用了`swtch`函数，现在的恢复执行会从自己的`swtch`函数返回
- 返回之后，调度器线程会释放刚刚让出了CPU的进程的锁

在`xv6`中，不允许进程在执行`swtch`函数的过程中，持有任何其他的锁。即进程在调用`swtch`函数的时候，必须要持有`p->lock`，但又不能同时持有任何其他的锁。

---

## Sleep&Wakeup接口

有很多种情况可能导致进程需要等待一些特定事件，这些特定事件可能来自I/O，也可能来自其他进程。**Coordination**是帮助我们解决这些问题并实现这些需求的工具。一种最简单的方法就是循环并等待，有时**Coordination**会是让出CPU知道等待的事件发生。**Coordination**有很多种实现方式，在`xv6`中使用的是`Sleep&Wakeup`方式。

值得注意的是，`sleep`和`wakeup`中都有一个`chan`的参数，因为当调用`wakeup`函数时，我们只想唤醒正在等待刚刚发生的特定事件的线程，这个参数就是用来表明我们等待的特定事件的。另外，`sleep`还需要一个锁作为参数传入。

---

## Lost wakeup

假设我们有一个不需要锁作为参数的`sleep`实现，它接收`sleep channel`作为唯一的参数，那么它会导致的结果就是**lost wakeup**，这种实现称为**broken_sleep**，

**Lost Wakeup**是指在多线程/多进程环境中，当一个进程准备进入睡眠状态时，另一个进程的唤醒信号在该进程真正进入睡眠之前就已经发出，导致唤醒信号丢失，使得睡眠的进程永远无法被唤醒的问题。如下面的实现：

```c
// 有问题的实现
void uartwrite(char buf[], int n) {
    acquire(&uart_tx_lock);
    
    for(int i = 0; i < n; i++) {
        while(tx_done == 0) {
            release(&uart_tx_lock);  // 释放锁
            broken_sleep(tx_chan);    // 在这里有时间窗口！
            acquire(&uart_tx_lock);   // 重新获取锁
        }
        // 发送字符...
        tx_done = 0;
    }
    
    release(&uart_tx_lock);
}

void uartintr(void) {
    acquire(&uart_tx_lock);
    
    if(UART硬件完成发送) {
        tx_done = 1;
        wakeup(tx_chan);  // 可能在进程sleep之前就调用了
    }
    
    release(&uart_tx_lock);
}
```

![[Drawing 2025-06-10 16.19.01.excalidraw]]

问题的根本原因在于`release(&uart_tx_lock)`和`broken_sleep(tx_chan)`之间存在一个时间窗口，而中断处理程序可能在这个窗口内执行，这样就可能导致`wakeup`在`sleep`前被调用。

因此为了保护共享数据（`tx_done`标志位）以及硬件访问同步（防止多个线程同时访问`UART`硬件寄存器），需要锁保护。

`xv6`通过原子性操作解决了这个问题：

```c
void sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // 关键：先获取进程锁
  acquire(&p->lock);  
  // 然后释放原来的锁
  release(lk);

  // 原子性地设置睡眠状态
  p->chan = chan;
  p->state = SLEEPING;

  sched();  // 切换到调度器

  // 清理工作
  p->chan = 0;
  
  // 重新获取原来的锁
  release(&p->lock);
  acquire(lk);
}
```

![[Drawing 2025-06-10 16.29.43.excalidraw]]

---

## Pipe中的sleep和wakeup

当`read`系统调用到`piperead`函数时，`pi->lock`会用来保护`pipe`，这个就是`sleep`函数对应的条件锁，`piperead`需要等待的条件是`pipe`中有数据，即`pi->nwrite > pi->nread`，如果这个条件不满足，`piperead`就会调用`sleep`并等待，同时它会讲`pi->lock`作为参数传递给`sleep`函数，以免发生**lost wakeup**。

```c
int
piperead(struct pipe *pi, uint64 addr, int n)
{
  int i;
  struct proc *pr = myproc();
  char ch;

  acquire(&pi->lock);
  while(pi->nread == pi->nwrite && pi->writeopen){  //DOC: pipe-empty
    if(killed(pr)){
      release(&pi->lock);
      return -1;
    }
    sleep(&pi->nread, &pi->lock); //DOC: piperead-sleep
  }
  for(i = 0; i < n; i++){  //DOC: piperead-copy
    if(pi->nread == pi->nwrite)
      break;
    ch = pi->data[pi->nread++ % PIPESIZE];
    if(copyout(pr->pagetable, addr + i, &ch, 1) == -1)
      break;
  }
  wakeup(&pi->nwrite);  //DOC: piperead-wakeup
  release(&pi->lock);
  return i;
}
```

`pipewrite`会向`pipe`的缓存写数据，并最后在`piperead`所等待的`sleep channel`上调用`wakeup`。我们希望避免在`piperead`检查发现没有字节可以读取到它调用`sleep`函数之间，另一个CPU调用`pipewrite`函数，因为这会产生一次**lost wakeup**。

```c
int
pipewrite(struct pipe *pi, uint64 addr, int n)
{
  int i = 0;
  struct proc *pr = myproc();

  acquire(&pi->lock);
  while(i < n){
    if(pi->readopen == 0 || killed(pr)){
      release(&pi->lock);
      return -1;
    }
    if(pi->nwrite == pi->nread + PIPESIZE){ //DOC: pipewrite-full
      wakeup(&pi->nread);
      sleep(&pi->nwrite, &pi->lock);
    } else {
      char ch;
      if(copyin(pr->pagetable, &ch, addr + i, 1) == -1)
        break;
      pi->data[pi->nwrite++ % PIPESIZE] = ch;
      i++;
    }
  }
  wakeup(&pi->nread);
  release(&pi->lock);

  return i;
}
```

在`pipe`的代码中，`pipewrite`和`piperead`都将`sleep`包装在一个`while`循环中。`piperead`中的循环等待`pipe`的缓存为非空，`pipewrite`中的循环等待`pipe`的缓存不为`full`。之所以要这么做，是因为可能有多个进程在读取同一个`pipe`。

如果一个进程向`pipe`中写入一个字节，这个进程会调用`wakeup`进而同时唤醒所有在读取同一个`pipe`的进程。

在调用`sleep`函数的时候，需要对`condition lock`上锁，在`sleep`函数中会解锁，然后在返回时会重新对它上锁。这样第一个被唤醒的进程的内核线程会持有`condition lock`，而其他的线程在重新对`condition lock`上锁的时候在锁的`acquire`函数中等待。

而这个获取到锁的进程会从`sleep`函数中返回，它检查到`pipe`中有一个字节并从循环中退出、读取，然后`piperead`释放锁并返回。

接下来到第二个被唤醒的进程的内核线程，它的`sleep`函数可以获取`condition lock`并返回，但检查发现没有可以读的，所以它与其他所有等待线程都会重新进入`sleep`函数。

![[Drawing 2025-06-10 17.00.16.excalidraw]]

---

## exit和wait系统调用

接下来讨论另一个与`Sleep&Wakeup`相关的挑战：如何关闭一个进程。在`xv6`中，一个进程如果要退出，需要释放用户内存，释放`page table`、`trapframe`对象，将进程在进程表单标为`REUSABLE`。

这里会产生两大问题：
- 首先我们不能单方面直接摧毁一个线程，因为另一个线程可能正在另一个CPU核上运行，并使用着自己的栈；也可能另一个线程正在内核中持有锁；或者是正在更新一个复杂的内核数据。
- 即使一个线程调用了`exit`，仍然需要一种方法让线程释放最后的几个对于运行代码来说关键的资源。

在`xv6`中，与之相关的有`exit`和`kill`函数。先看`exit`函数。

```c
// Exit the current process.  Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // Close all open files.
  for(int fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd]){
      struct file *f = p->ofile[fd];
      fileclose(f);
      p->ofile[fd] = 0;
    }
  }

  begin_op();
  iput(p->cwd);
  end_op();
  p->cwd = 0;

  acquire(&wait_lock);

  // Give any children to init.
  reparent(p);

  // Parent might be sleeping in wait().
  wakeup(p->parent);
  
  acquire(&p->lock);

  p->xstate = status;
  p->state = ZOMBIE;

  release(&wait_lock);

  // Jump into the scheduler, never to return.
  sched();
  panic("zombie exit");
}
```

`exit`最后会释放进程的内存和`page table`，关闭已经打开的文件，同时父进程会从`wait`调用中被唤醒。

它首先关闭自己拥有的文件，然后进程有一个对当前目录的记录，`exit`也需要将对这个目录的引用释放给文件系统。如果要退出的进程还有子进程，接下来需要设置这些子进程的父进程为`init`进程。每一个正在`exit`的进程，都有一个父进程中的对应`wait`系统调用，这个系统调用会完成进程退出的最后几个步骤。所以需要设置这些子进程的父进程为`init`，即`PID`为1的进程。

之后，调用`wakeup`唤醒当前进程的父进程，设置当前进程的状态为`ZOMBIE`。现在进程还没有完全释放它的资源，因此不能被重用。

![[Drawing 2025-06-10 17.52.34.excalidraw]]

接下来会进入父进程的`wait`系统调用的返回，它的返回表明当前进程的一个子进程退出了。

```c
// Wait for a child process to exit and return its pid.
// Return -1 if this process has no children.
int
wait(uint64 addr)
{
  struct proc *pp;
  int havekids, pid;
  struct proc *p = myproc();

  acquire(&wait_lock);

  for(;;){
    // Scan through table looking for exited children.
    havekids = 0;
    for(pp = proc; pp < &proc[NPROC]; pp++){
      if(pp->parent == p){
        // make sure the child isn't still in exit() or swtch().
        acquire(&pp->lock);

        havekids = 1;
        if(pp->state == ZOMBIE){
          // Found one.
          pid = pp->pid;
          if(addr != 0 && copyout(p->pagetable, addr, (char *)&pp->xstate,
                                  sizeof(pp->xstate)) < 0) {
            release(&pp->lock);
            release(&wait_lock);
            return -1;
          }
          freeproc(pp);
          release(&pp->lock);
          release(&wait_lock);
          return pid;
        }
        release(&pp->lock);
      }
    }

    // No point waiting if we don't have any children.
    if(!havekids || killed(p)){
      release(&wait_lock);
      return -1;
    }
    
    // Wait for a child to exit.
    sleep(p, &wait_lock);  //DOC: wait-sleep
  }
}
```

它里面包含了一个大的循环，这个循环扫描进程表单，找到父进程是自己且状态为`ZOMBIE`的进程。父进程会调用`freeproc`函数来完成释放进程资源的最后几个步骤。

```c
// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

这是关闭进程的最后一个步骤，它释放`trapframe`和`page table`。

![[Drawing 2025-06-11 14.03.48.excalidraw]]

`wait`是进程退出的重要组成部分，`init`进程的工作就是在一个循环中不断调用`wait`，因为每个进程都需要对应一个`wait`，这样才能调用`freeproc`函数并清理进程的资源。

当父进程完成了进程所有资源的清理，子进程的状态会被设置为`UNUSED`，之后`fork`系统调用才能重用进程在进程表单的位置。

---

## kill系统调用

最后来看`kill`系统调用，Unix中的一个进程可以将另一个进程的ID传递给`kill`系统调用，并让另一个进程停止运行。但这样有可能会`kill`掉一个还在内核执行代码的进程，因此我们不能让`kill`直接停止目标进程的运行。

```c
// Kill the process with the given pid.
// The victim won't exit until it tries to return
// to user space (see usertrap() in trap.c).
int
kill(int pid)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++){
    acquire(&p->lock);
    if(p->pid == pid){
      p->killed = 1;
      if(p->state == SLEEPING){
        // Wake process from sleep().
        p->state = RUNNABLE;
      }
      release(&p->lock);
      return 0;
    }
    release(&p->lock);
  }
  return -1;
}
```

它先扫描进程表单，找到目标进程，然后将进程的`proc`结构体中`killed`标志位设置为1，如果进程在`SLEEPING`状态则将其设置为`RUNNABLE`，`kill`系统调用本身是很温和的。

而目标进程运行到内核代码中能安全停止运行的位置，会检查自己的`killed`标志位，如果设置为1，目标进程会自愿执行`exit`系统调用。在这个位置，代码没有持有任何锁，也不执行任何操作，因此通过`exit`退出是完全安全的。

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

而如果进程在`SLEEPING`状态被`kill`了，它会实际地退出。可以看到上面`kill`代码中如果进程是`SLEEPING`状态，它会将其设置为`RUNNABLE`状态。这意味着调度器会重新运行进程，并且进程会从`sleep`中返回。

```c
int
piperead(struct pipe *pi, uint64 addr, int n)
{
  int i;
  struct proc *pr = myproc();
  char ch;

  acquire(&pi->lock);
  while(pi->nread == pi->nwrite && pi->writeopen){  //DOC: pipe-empty
    if(killed(pr)){
      release(&pi->lock);
      return -1;
    }
    sleep(&pi->nread, &pi->lock); //DOC: piperead-sleep
  }
  for(i = 0; i < n; i++){  //DOC: piperead-copy
    if(pi->nread == pi->nwrite)
      break;
    ch = pi->data[pi->nread++ % PIPESIZE];
    if(copyout(pr->pagetable, addr + i, &ch, 1) == -1)
      break;
  }
  wakeup(&pi->nwrite);  //DOC: piperead-wakeup
  release(&pi->lock);
  return i;
}
```

如果一个进程正在`sleep`状态等待从`pipe`中读取数据，然后它被`kill`了，然后被`kill`函数重启，从`sleep`中返回到循环的最开始，这时`pipe`中大概率依然没有数据，在`piperead`中会判断进程是否被`kill`了，然后它会返回`-1`，并回到`usertrap`函数的`syscall`位置。

之后在`usertrap`函数中会检查`p->killed`并调用`exit`。

当然也有一些情况，如果进程在`SLEEPING`状态中被`kill`了并不能直接退出。例如一个进程正在更新一个文件系统并创建一个文件。磁盘驱动中的`sleep`循环就不会检查进程的`killed`标志位。

在**Linux**或者真正的操作系统中，每个进程都有一个`user_id`对应执行进程的用户，一些系统调用使用进程的`user_id`来检查进程允许做的操作。在**Linux**中会有额外的检查，调用`kill`的进程必须与被`kill`的进程有相同的`user_id`，否则`kill`操作不被允许。

`init`进程不会退出，它就是在一个循环中不停调用`wait`，如果它退出，就是一个错误。