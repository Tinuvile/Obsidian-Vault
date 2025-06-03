# Scheduling

任何操作系统都可能运行比计算机有的CPU更多的进程，因此需要一个计划来在进程间分时共享CPU。一种常见的方法是将进程多路复用到硬件CPU上，为每个进程提供拥有自己虚拟CPU的假象。

---

## Multiplexing

## 多路复用

`xv6`通过在两种情况下将每个CPU从一个进程切换到另一个进程来进行多路复用。首先，`sleep`和`wakeup`机制会在进程等待设备或管道I/O完成、等待子进程退出或在`sleep`系统调用中等待时进行切换；其次，`xv6`定期强制切换以处理长时间不计算且不睡眠的进程。

多路复用的实现需要考虑多个问题：
- 如何从一个进程切换到另一个进程？
- 如何以对用户进程透明的方式强制切换？
- 许多CPU可能同时在不同进程间切换，需要一个锁来避免竞争；
- 进程退出时需释放其内存和其他资源，但需要外部完成；
- 多核机器的每个核心必须记住它在执行哪个进程；
- `sleep`和`wakeup`允许一个进程放弃CPU并睡眠等待事件，并允许另一个进程唤醒该进程，但需小心避免会导致唤醒通知丢失的竞争。
![[Pasted image 20250528134153.png]]

---

## Code：Context switching

上图概述了从一个用户进程切换到另一个用户进程所涉及的步骤：
- 首先通过系统调用或者中断，从用户空间转换到旧进程的内核空间；
- 然后上下文切换到当前CPU的调度器线程；
- 再切换到新进程的内核线程；
- 最后`trap`返回到用户空间。

`xv6`调度器为每个CPU都有一个专用线程（有保存的寄存器和堆栈）。在旧进程的内核堆栈上执行调度器不安全，因为其他核心也可能唤醒该进程并运行，这将导致在不同的核心上使用相同的堆栈。

另外，切换线程还涉及到保存旧线程的CPU寄存器，以及恢复新进程之前保存的寄存器。恢复栈指针和程序计数器就意味着CPU将切换栈并切换它正在执行的代码。函数`swtch`负责执行内核线程切换的保存和恢复操作，它会保存和恢复寄存器集（称为上下文）。当一个进程需要让出CPU时，它的内核线程会调用`swtch`来保存自己的上下文并返回到调度器上下文。每个上下文都包含在一个`struct context`中，而它本身又包含在进程的`struct proc`中或 CPU 的`struct cpu`中。

```c
// Saved registers for kernel context switches.
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

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

`swtch`接收两个参数`struct context *old`和`struct context *new`。它将当前寄存器保存到`old`中，再从`new`中加载寄存器，然后返回。

现在假想跟随一个进程通过`swtch`进入调度程序。中断结束时的一种可能是`usertrap`调用`yield`，然后`yield`调用`sched`，`sched`调用`swtch`将当前上下文保存到`p->context`中，并切换到之前保存的调度程序上下文。

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

`swtch`函数自身（被调用者）仅保存*Callee-saved registers*，*Caller-saved registers*由调用函数（调用者）负责保存和恢复，即如果有需要保存的值，需要在调用函数之前将这些寄存器的值压入栈中，并在调用返回后恢复。

> [!faq] 责任划分机制
> 假设有以下C代码调用关系：
> ```c
> void funcA() { 
>    int x = 10; 
>    funcB(); // 调用funcB 
>    // 此时需要确保x的值仍然是10 
> } 
> void funcB() { 
>    // funcB可能会修改caller-saved寄存器
> }
> ```
> - 寄存器保存流程：
 >   1. `funcA`在调用`funcB`前，若需要保留某些 caller-saved 寄存器（如`eax`），则将其压栈。
 >   2. 调用`funcB`，`funcB`无需关心 caller-saved 寄存器的原始值。
 >   3. `funcB`在执行前保存 callee-saved 寄存器（如`ebp`），执行后恢复这些寄存器。
 >   4. `funcB`返回后，`funcA`从栈中恢复 caller-saved 寄存器的值。
> - 这种分工的主要目的是优化性能：
>     - Callee-saved 寄存器通常用于保存函数内部的长期状态（如局部变量），因此由被调用者管理更高效。
>     - Caller-saved 寄存器常用于短期数据传递（如函数参数），调用者可以更灵活地决定是否需要保存这些值。

`swtch`知道每个寄存器字段在`struct context`中的偏移量，它不保存程序计数器，但保存`ra`寄存器，该寄存器存有调用`swtch`的返回地址。

```assembly
swtch:
        # 保存当前进程的上下文到 p->context
        sd ra, 0(a0)      # 保存返回地址
        sd sp, 8(a0)      # 保存栈指针
        sd s0, 16(a0)     # 保存 callee-saved 寄存器
        sd s1, 24(a0)
        # ... 其他寄存器
        
        # 加载调度器的上下文 cpu->context  
        ld ra, 0(a1)      # 加载返回地址
        ld sp, 8(a1)      # 加载栈指针
        ld s0, 16(a1)     # 加载 callee-saved 寄存器
        ld s1, 24(a1)
        # ... 其他寄存器
        
        ret               # 返回到 ra 指向的地址
```

`swtch`会从新的上下文中恢复寄存器，这些寄存器保存了之前`swtch`保存的寄存器值。当`swtch`返回时，它返回到恢复的`ra`寄存器指向的指令，即返回到调用`swtch`的下一条指令。不过，它会在新线程的栈上返回。

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

在这个流程中，单独再来看看返回地址的处理：
1. 第一次调用：`swtch(&p->context, &mycpu->context)`：
   - 保存当前进程的`ra`（指向`sched()`中`swtch`的下一条指令）
   - 加载调度器的`ra`（指向`scheduler()`中`swtch`的下一条指令）
   - 返回到调度器继续执行
1. 第二次调用：`swtch(&c->context, &p->context)`：
   - 保存调度器的`ra`（指向`scheduler()`中`swtch`的下一条指令）
   - 加载进程的`ra`（指向`sched()`中`swtch`的下一条指令）
   - 返回到进程的`sched()`继续执行

```text
进程A执行中
    ↓
yield() 
    ↓
sched()
    ↓
swtch(&p->context, &mycpu()->context)  → 保存进程A上下文，加载调度器上下文
    ↓                                    
scheduler() 的 swtch 返回点              ← 跳转到这里
    ↓
调度器寻找可运行进程B
    ↓
swtch(&c->context, &p->context)        → 保存调度器上下文，加载进程B上下文
    ↓
进程B的 sched() 中的 swtch 返回点        ← 跳转到这里
    ↓
进程B从yield()返回，继续执行
```

> [!faq] FAQ
> - 返回点是什么？
>   这里的返回点不是函数返回，而是指令执行位置的恢复，即`ra`寄存器指向的地址。比如进程调用`sched()`：
>   ```c
>   // 在sched()函数中（第503行）
>   intena = mycpu()->intena;
>   swtch(&p->context, &mycpu()->context);  // ← 调用位置
>   mycpu()->intena = intena;               // ← 这里是"返回点"！
>   ```
>   当进程执行到 swtch 调用时：
>   1. 调用指令执行前：`ra`寄存器被自动设置为第504行的地址
>   2. `swtch`保存上下文：将这个`ra`值保存到`p->context.ra`
>   3. 加载调度器上下文：从`mycpu()->context.ra`加载新的`ra`
>   4. `ret`指令执行：跳转到新`ra`指向的地址
> - `mycpu()->context`为什么是调度器上下文？
>   查看`main()`函数，可以看到每个CPU核心在启动时都直接调用了`scheduler()`：
>   ```c
>   void main() {
>     // ... 各种初始化
>     scheduler();  // 直接调用，永不返回
>   }
>   ```
>   故而调度器是在CPU的原始上下文中开始运行的，它的栈在内核栈中。
> - `scheduler()`的`swtch`返回点
>   ```c
>   c->proc = 0;
>   found = 1;
>   ```
> - 进程B的`sched()`中的`swtch`返回点
>   这里的关键在于：**进程B不是第一次运行！**
>   可以查看`proc.c`中新进程的初始化：
>   ```c
>   // Set up new context to start executing at forkret
>   memset(&p->context, 0, sizeof(p->context));
>   p->context.ra = (uint64)forkret;  // 新进程第一次运行时会到forkret
>   p->context.sp = p->kstack + PGSIZE;
>   ```
>   进程B第一次被调度是从`forkret`开始执行。然后在后续的某个时刻它调用`yield()->sched()->swtch()`让出CPU，而现在调度器重新选中进程B，`swtch`将返回到进程B在`sched()`中的返回点，也即问题1中那个例子的位置。

确实精妙！这几个函数之间的关系。

---

## Code：Scheduling

上一节讨论了`swtch`的工作细节，在这一节中只需要将`swtch`抽象一下即可，主要研究从一个进程的内核线程通过调度器切换到另一个进程的过程。

调度器的存在形式是每个CPU中的一个特殊线程，每个线程都会运行调度器函数，该函数会负责选择接下来要运行的进程。任何想放弃CPU的进程必须获取自己的进程锁`p->lock`，然后释放它持有的任何其他锁，并更新自己的状态`p->state`，调用`sched`。`yield()`、`sleep()`和`exit()`这些函数都遵循这一步骤。这些在`sched`中也会再次检查。

然后因为持有锁，中断会被禁用。最后，`sched`调用`swtch`将当前上下文保存在`p->context`中，并切换到`cpu->scheduler`中的调度器上下文。`swtch`在调度器的栈上返回，调度器寻找要运行的进程，然后切换。

值得注意的是，`xv6`在调用`swtch`时持有`p->lock`，`swtch`的调用者必须已经持有锁，并且锁的控制权会传递给切换到的代码。这并不常见，因为大多数获取锁的线程也会负责释放锁。不过在上下文切换中，因为锁保护的进程状态和上下文字段的不变量并不成立，因此这样是必要的。否则可能出现：在`yield()`将进程状态设置为`RUNNABLE`之后，但在`swtch`导致它停用自己的内核栈前，另一个CPU决定运行该进程，这样就会出现错误。

在`xv6`中，内核线程总是通过`sched()`函数中的`swtch()`一行让出CPU，又总是恢复于下一行；而调度器总是在`scheduler()`函数的`c->proc=p`一行让步，又在`swtch()`一行恢复。

```text
调度器协程 (scheduler)                 进程协程 (sched)
     │                                      │
     ├─ 第465行: swtch(&c->context, &p->context)
     │              │                       │
     │              └────────────────────►  │
     │                                      │
     │ ◄─────────── 暂停在第466行 ───────────┤ 第505行: swtch(&p->context, &mycpu()->context)
     │                                      │              │
     │                                      │              │
     │  第466行恢复: c->proc = 0; ◄──────────┤ ◄────────────┘
     │                                      │
     │              (寻找下一个进程)          │ 暂停在第506行
     │                                      │
     ├─ 再次第465行: swtch(...) ──────────►  │ 第506行恢复: mycpu()->intena = intena;
     │                                      │
```

> [!info] *coroutines*
> 这种在两个线程之间进行风格化切换的过程有时被称为协程。在上面的例子中，`sched`和`scheduler`就是彼此的协程。

不过有一种情况，调度器调用`swtch`不会在`sched`中结束。当一个新进程首次被调度时，它从`forkret`开始。`forkret`的存在是为了释放`p->lock`，否则，新进程可以从`usertrapret`开始。

> [!faq] 关于`forkret`
> 它的关键作用在于处理锁的所有权转移。
> 在`scheduler()`函数中：
> ```c
> for(p = proc; p < &proc[NPROC]; p++) {
> acquire(&p->lock);                    // ⭐ 调度器获取进程锁
> if(p->state == RUNNABLE) {
>   p->state = RUNNING;
>   c->proc = p;
>   swtch(&c->context, &p->context);    // ⭐ 切换到进程，但锁仍被持有！
> ```
> 调度器在切换到进程之前获取了锁，然后在`swtch`后，锁的所有权转移给了新进程。
> 新进程在第一次调度前的初始化前面也说到了，会在`forkret`开始执行，但是继承了调度器持有的`p->lock`。
> ```c
> // A fork child's very first scheduling by scheduler()
> // will swtch to forkret.
> void
> forkret(void)
> {
>   static int first = 1;
>  
>   // Still holding p->lock from scheduler.
>   release(&myproc()->lock);
>  
>   if (first) {
>     // File system initialization must be run in the context of a
>     // regular process (e.g., because it calls sleep), and thus cannot
>     // be run from main().
>     fsinit(ROOTDEV);
>  
>     first = 0;
>     // ensure other cores see first=0.
>     __sync_synchronize();
> }
> ```
> 在`forkret`中有释放操作。如果直接从`usertrapret()`开始，返回用户空间并执行程序，那就可能有死锁、调度器阻塞等等风险。

调度程序运行循环遍历进程表，寻找一个可运行的程序，即`p->state == RUNNABLE`。一旦找到，它就会设置每个CPU的当前进程变量`c->proc`，将进程标记为`RUNNING`，然后调用`swtch`开始运行。

也可以从不变量的角度来思考调度代码的结构与顺序，它强制指向关于每个进程的一组不变量，并在这些不变量不成立时立即持有`p->lock`。

> [!info] *invariants*
> 不变量是系统必须始终满足的条件，在`xv6`中，每个进程状态都对应特定的不变量，当这些条件被暂时破坏时，必须持有`p->lock`来防止并发访问。

比如`RUNNING`状态就是一个不变量，如果一个进程是`RUNNING`状态，定时器中断的`yield()`必须能安全地切换走该进程。这意味着在此刻，CPU寄存器必须保存有该进程的寄存器值（即`swtch`还没有将它们移动到`context`中），且`c->proc`必须指向该进程。代码体现在`scheduler()`：

```c
p->state = RUNNING;
// ⚠️ 如果这里发生时钟中断，c->proc 还是旧值！
c->proc = p;
swtch(&c->context, &p->context);  // 此时不变量成立
```

`RUNNABLE`状态也是一个不变量，如果进程是`RUNNABLE`状态，空闲CPU的调度器必须能够安全地运行它。这意味着`p->context`必须保存进程的寄存器（它们不在真实寄存器中），没有CPU在该进程的内核栈上执行，且没有CPU的`c->proc`指向该进程。代码体现在`yield()`：

```c
void yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;    // 设置状态
  sched();                // 调用swtch保存寄存器到p->context
  release(&p->lock);
}
```

而这些不变量在持有`p->lock`时经常不成立。

```c
acquire(&p->lock);           // 🔒 获取锁
if(p->state == RUNNABLE) {
  // ⚠️ 不变量被暂时破坏的时期
  p->state = RUNNING;        // 状态改变，但 c->proc 还未更新
  c->proc = p;               // c->proc 更新，但寄存器还在 p->context 中
  swtch(&c->context, &p->context);  // 寄存器被恢复，不变量重新成立
  // 🔓 锁的所有权转移给进程
```

```c
// 进程持有 p->lock
p->state = RUNNABLE;         // ⚠️ 状态改变，但寄存器还在CPU中
swtch(&p->context, &mycpu()->context);  // 寄存器被保存，不变量重新成立
// 🔓 锁的所有权转移给调度器
```

维护这种不变性也是为什么`xv6`经常在一个线程中获取`p->lock`并在另一个线程中释放它的原因。

---

## Code：mycpu and myproc

`xv6`经常需要一个指向当前进程的`proc`结构的指针。它为每个CPU维护一个`struct cpu`，这个结构记录了当前在该CPU上运行的进程、CPU调度器线程的保存寄存器以及管理中断禁用所需的嵌套自旋锁计数。

```c
// Per-CPU state.
struct cpu {
  struct proc *proc;          // The process running on this cpu, or null.
  struct context context;     // swtch() here to enter scheduler().
  int noff;                   // Depth of push_off() nesting.
  int intena;                 // Were interrupts enabled before push_off()?
};
```

`mycpu()`返回指向当前CPU的`struct cpu`的指针，RISC-V会为每个CPU编号，给每个CPU一个`hartid`，`xv6`则会确保在内核中运行时，每个CPU的`hartid`都存储在该CPU的`tp`寄存器中。这使得`mycpu()`可以使用`tp`来索引一个`cpu`结构数组。实际上这样的机制根本在于RISC-V架构，监督模式无法直接读取`hartid`，只能用`tp`寄存器当“中转站”。

在CPU启动序列的早期，仍在机器模式下，`mstart`函数会设置`tp`：

```c
// keep each CPU's hartid in its tp register, for cpuid().
int id = r_mhartid();  // 在机器模式下读取真实hartid
w_tp(id);              // 保存到tp寄存器
```

然后返回用户空间前，在`trampoline`页中保存内核的`tp`值：

```c
// usertrapret() 函数中
p->trapframe->kernel_hartid = r_tp();  // 保存当前tp到trapframe
```

这是因为用户进程可能会修改`tp`寄存器。

最后，`uservec`从用户空间返回内核时恢复保存的`tp`寄存器：

```assembly
# uservec 中的关键代码
# make tp hold the current hartid, from p->trapframe->kernel_hartid
ld tp, 32(a0)  # 从trapframe恢复tp寄存器
```

一个完整的生命周期如下：

```text
启动阶段 (机器模式):
mstart() → r_mhartid() → w_tp(hartid) → 切换到监督模式

内核运行阶段 (监督模式):
cpuid() → r_tp() → 返回当前CPU ID

用户空间切换 (监督模式):
usertrapret() → 保存tp到trapframe → userret() → 用户程序运行
                                      ↓
用户程序 (可能修改tp) → 系统调用/中断 → uservec() → 恢复tp
```

但`cpuid`和`mycpu`的返回值仍然有可能出错，比如定时器中断并导致线程让出CPU，然后移动到另一个CPU，那么先前返回的值将不再正确。

```c
// 错误代码示例
int my_cpu_id = cpuid();           // 假设返回CPU 0
struct cpu *c = &cpus[my_cpu_id];  // 获取CPU 0的结构体
// ⚠️ 这里发生时钟中断！进程被调度到CPU 1
c->some_field = value;             // 💥 错误！修改了CPU 0的数据，但实际运行在CPU 1上
```

为避免这个问题，`xv6`要求调用者禁用中断，并且只有在使用完返回的`struct cpu`后才重新启用中断。

```c
struct proc*
myproc(void)
{
  push_off();                    // ⭐ 关闭中断
  struct cpu *c = mycpu();       // 安全地获取当前CPU
  struct proc *p = c->proc;      // 获取当前进程
  pop_off();                     // ⭐ 恢复中断
  return p;
}
```

---

## Sleep and wakeup

前面的调度和锁都是在给另一个进程隐藏本进程的存在，这个机制则是一个帮助进程进行有意交互的抽象。它允许一个进程在等待事件时睡眠，而另一个进程在事件发生后唤醒它。*Sleep and wakeup*通常被称作**sequence coordination**或者**conditional synchronization mechanisms**。

首先考虑一种称为*semaphore*的同步机制，它协调生产者和消费者。信号量维护一个计数并提供两个操作：V操作针对生产者，增加计数；P操作针对消费者，等待直到计数非零，然后递减计数并返回。如果只有一个生产者线程和一个消费者线程，并且执行在不同的CPU上，则可采用下面的实现：

```python
struct semaphore {
  struct spinlock lock;
  int count;
}

void
V(struct semaphore *s)
{
   acquire(&s->lock);
   s->count += 1;
   release(&s->lock);
}

void
P(struct semaphore *s)
{
   while(s->count == 0);
   acquire(&s->lock);
   s->count -= 1;
   release(&s->lock);
}
```

但上面实现的开销很大。 如果生产者很少行动，消费者将花费大部分时间在`while`循环中自旋。避免这个需要一种方法让消费者让出CPU，并在`V`增加`count`后恢复执行。考虑`sleep`与`wakeup`调用。`sleep(chan)`使进程睡眠并释放CPU，然后该进程被加入`chan`对应的等待通道中。`wakeup(chan)`唤醒所有在`chan`上睡眠的进程，使它们的`sleep`调用返回，如果`chan`中没有进程等待，则不做任何操作。现在可以修改信号量的实现：

```c
void
V(struct semaphore *s)
{
   acquire(&s->lock);
   s->count += 1;
   wakeup(s);
   release(&s->lock);
}

void
P(struct semaphore *s)
{
   while(s->count == 0)
     sleep(s);
   acquire(&s->lock);
   s->count -= 1;
   release(&s->lock);
}
```

但现在这种设计依然有问题，假设`P`发现`s->count == 0`但还没有进入下一行，而`V`运行在另一个CPU，它将`s->count`改为非零并调用`wakeup`，而`wakeup`会发现没有进程在睡眠，因此什么也不做，这时`P`调用`sleep`进入睡眠，这样的话`P`正在等待一个已经发生的`V`调用，并可能永远等待。

该问题的根源在于，**`P`只在`s->count == 0`时睡眠**这一不变量被`V`在错误时刻的运行破坏了。保护这个不变量的方法是将`P`中锁的获取移动位置。

```c
void
V(struct semaphore *s)
{
   acquire(&s->lock);
   s->count += 1;
   wakeup(s);
   release(&s->lock);
}

void
P(struct semaphore *s)
{
   acquire(&s->lock);
   while(s->count == 0)
     sleep(s);
   s->count -= 1;
   release(&s->lock);
}
```

现在这个版本又可能导致死锁，`P`在睡眠时持有锁。`V`将永远阻塞等待锁。通过修改`sleep`的接口可以修复这个问题。调用者必须将锁传递给`sleep`，这样在调用进程被标记为睡眠并在睡眠通道上等待后，`sleep`就可以释放锁。这把锁会强制一个并发的`P`进程进入睡眠状态，以便`wakeup`能等待`P`将自身设置为睡眠，然后`wakeup`会找到处于睡眠状态的消费者将其唤醒。一旦消费者再次醒来，`sleep`会在返回之前重新获取该锁。

```c
void
V(struct semaphore *s)
{
   acquire(&s->lock);
   s->count += 1;
   wakeup(s);
   release(&s->lock);
}

void
P(struct semaphore *s)
{
   acquire(&s->lock);
   while(s->count == 0)
     sleep(s, &s->lock);
   s->count -= 1;
   release(&s->lock);
}
```

---

## Code：Sleep and wakeup

现在来看`sleep`和`wakeup`的实现。

```c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.

  acquire(&p->lock);  //DOC: sleeplock1
  release(lk);

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  release(&p->lock);
  acquire(lk);
}

// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    if(p != myproc()){
      acquire(&p->lock);
      if(p->state == SLEEPING && p->chan == chan) {
        p->state = RUNNABLE;
      }
      release(&p->lock);
    }
  }
}
```

`sleep`把当前进程标记为`SLEEPING`，然后调用`sched`释放CPU，`wakeup`查找给定等待通道上睡眠的进程并将其标记为`RUNNABLE`，`sleep`和`wakeup`的调用者可以使用任何方便的数字作为通道，`xv6`通常使用参与等待的内核数据结构的内存地址。

`sleep`获取`p->lock`后，这个即将进入睡眠的进程同时持有`p->lock`和`lk`，`P`进程持有`lk`可以确保`V`进程无法调用`wakeup(chan)`。在`P`持有`p->lock`后就可以释放`lk`了，其他进程可能会调用`wakeup(chan)`，但`wakeup`会等待获取`p->lock`，这样它会等待`sleep`完成进程睡眠，从而避免`wakeup`错过`sleep`。

但如果`lk`和`p->lock`是一个锁，那么`sleep(p, &p->lock)`在获取`p->lock`时会与自身发生死锁。如果调用`sleep`的进程已经持有`p->lock`，它也不需要做额外的事来避免错过并发的`wakeup`，它可以记录睡眠通道，将进程状态改为`SLEEPING`并调用`sched`来使进程进入睡眠状态。

即使有多个进程在同一通道上休眠，也没有问题，其他进程尽管会被唤醒，但它们会发现没有数据可读，这次唤醒是“虚假的”，它们也会再次休眠。而如果两个使用`sleep`和`wakeup`的操作意外选择了相同的通道，也没有问题，与上面一样它们会看到“虚假”的唤醒。

`sleep/wakeup`的魅力在于它即轻量级（无需创建特殊的数据结构作为睡眠通道），又提供了一层间接性（调用者无需知道它们正在与哪个特定进程交互）。

---

## Code：Pipes

`pipe`也使用`sleep`和`wakeup`来同步生产者和消费者。每个管道由一个`struct pipe`表示，其中包含一个锁和一个数据缓冲区。字段`nread`和`nwrite`分别统计从缓冲区读取和写入的总字节数。缓冲区是循环的：在`buf[PIPESIZE - 1]`后写入的下一个字节是`buf[0]`。计数不会循环。这样可以区分满缓冲区`nwrite == nread + PIPESIZE`和空缓冲区`nwrite == nread`，但也意味着索引缓冲区时必须用`buf[nread % PIPESIZE]`。

```c
struct pipe {
  struct spinlock lock;
  char data[PIPESIZE];
  uint nread;     // number of bytes read
  uint nwrite;    // number of bytes written
  int readopen;   // read fd is still open
  int writeopen;  // write fd is still open
};
```

假设在两个不同的CPU上同时调用`piperead`和`pipewrite`，`pipewrite`先获取管道的锁，该锁保护计数、数据以及其他相关的不变量。`piperead`随后也尝试获取锁但失败，它会在`acquire`中自旋等待。同时，`pipewrite`循环遍历要写入的字节并一次将它们添加到管道中，在这个过程中可能发生缓冲区填满的情况。这时`pipewrite`会调用`wakeup`来提醒任何睡眠的管道读者缓冲区中有数据等待，然后它在`&pi->nwrite`上睡眠，等待读者从缓冲区中取出一些字节。`sleep`将`pipewrite`置于睡眠状态时释放`pi->lock`。

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

这时`pi->lock`可用，`piperead`会获取它并进入其临界区，然后执行读取将数据从管道中复制出来。这时有字节可供写入，`piperead`会在返回之前调用`wakeup`来唤醒任何睡眠的的写者。`wakeup`找到在`&pi->nwrite`上睡眠的进程并将该进程标记为`RUNNABLE`。

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

管道代码为读取者和写入者使用了独立的睡眠通道`pi->nread`和`pi->nwrite`，这在一些极端情况下会使系统更加高效，同时“并发”状态也会与上一章一样，“虚假“唤醒而继续睡眠。

---

## Code：Wait、exit and kill

睡眠与唤醒可以用于多种等待场景。一个例子是子进程`exit`和父进程`wait`之间的交互。

在子进程终止时，父进程可能已经处于中睡眠`wait`状态、或者正在执行其他操作。在后一种情况中，后续的`wait`调用必须能感知到子进程的终止，即便可能是在子进程调用`exit`很久之后。`xv6`记录子进程终止状态直到`wait`检测到它的方式是，让`exit`将调用者置于`ZOMBIE`状态，直到父进程的`wait`函数察觉该状态，将子进程状态改为`UNUSED`，复制子进程的退出状态，并向父进程返回子进程的进程ID。若父进程先于子进程退出，系统会将子进程转交给`init`进程，而`init`进程会持续调用`wait`函数，因此每个子进程最终都会有一个父进程来完成清理工作。这种机制的主要实现挑战在于：父进程与子进程的`wait`和`exit`操作之间，以及`exit`操作互相之间可能出现的竞争条件和死锁问题。

`wait`函数使用调用进程的`p->lock`作为条件锁，以避免唤醒丢失，并会在函数开始时获取该锁。随后，它扫描进程表：
- 如果找到一个处于`ZOMBIE`状态的子进程，它会释放该子进程的资源和`proc`结构体，然后将子进程的退出状态复制到`wait`传入的地址，并返回子进程的进程ID。
- 如果`wait`找到一个子进程但没有退出，它会调用`sleep`等待其中某个子进程退出，然后重新扫描。这里`sleep`释放的条件锁是等待进程的`p->lock`。`wait`通常持有两个锁，它会先获取自己的锁，然后尝试获取子进程的锁。因此`xv6`的所有相关操作必须遵循相同的加锁顺序（先父进程，后子进程）以避免死锁。

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

`wait`通过遍历每个进程的`np->parent`字段来查找子进程。它访问`np->parent`时无需持有`np->lock`，这看似违反了“共享变量必须由锁保护”的规则，但是在此场景下是安全的，原因在于：
- 进程的`parent`字段仅由其父进程修改。
- 若`np->parent == p`成立，除非当前进程主动修改，否则它的值不会改变。
- 若`np`是当前进程的祖先，获取`np->lock`可能因为违反“先父后子”的加锁顺序而导致死锁，因此不持有锁访问`parent`字段也是必要的设计妥协。

`exit`函数负责记录退出状态、释放部分资源、将子进程过继给`init`进程、唤醒可能处于`wait`状态的父进程、将自身标记为`ZOMBIE`、并永久让出CPU。

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

在将自身状态设置为`ZOMBIE`并唤醒父进程时，必须持有父进程的锁，因为这把锁是防止唤醒丢失的条件锁，若不持有该锁，父进程可能无法正确感知子进程的退出状态。子进程还必须持有自己的`p->lock`，否则父进程可能会看到它处于`ZOMBIE`状态并在它仍在运行时释放它。锁的获取顺序也很重要，必须先获取父进程的锁再获取子进程的锁。

`exit`调用的唤醒函数只唤醒父进程，且只有父进程在`wait`中睡眠时才唤醒。在设置`ZOMBIE`状态前唤醒父进程是安全的，因为父进程会在`wait`循环中等待子进程释放`p->lock`并完成状态更新后，才能正确扫描到子进程。

`kill`函数用于请求终止其他进程，但不会直接销毁目标进程。因为目标进程可能正在其他CPU上执行，且可能处于更新内核数据结构的敏感阶段。`kill`的核心操作仅仅是设置`p->killed`，并且唤醒目标进程（如果它处在睡眠状态）。
- 如果目标进程在用户空间运行，会通过系统调用或定时器中断进入内核，此时`usertrap`会检查`p->killed`标准并调用`exit`。
- 如果目标进程处于睡眠中，`kill`会调用`wakeup`使目标进程从`sleep`中返回，`xv6`的`sleep`调用通常包含在`while`循环中，返回后会重新检查条件和`p->killed`标志。若已设置，则进程会放弃当前操作，返回到`trap`，再次检查标志并退出。
- 还有一些特殊的`sleep`循环没有检查`p->killed`（比如`virtio`磁盘驱动），因为磁盘操作需保证原子性以维持文件系统一致性。此类进程会在完成当前系统调用后，才在`usertrap`中处理`killed`标志并退出。

---

## Real World

`xv6`调度器实现了一种简单的调度策略，即依次运行每个进程，这种策略称为*round robin*。实际的操作系统更加复杂，允许进程具有优先级，还需要保证公平性和高吞吐量等等。

睡眠和唤醒是一种简单有效的同步方法。但它面临的第一个挑战就是“丢失唤醒”问题。`Linux`内核的睡眠使用了一个显式的进程队列，称为等待队列，而不算等待通道，队列有自己的内部锁。

在`wakeup`中扫描整个进程列表以查找具有匹配`chan`的进程是低效的，更好的方法是将`sleep`和`wakeup`中的`chan`替换为一个数据结构，该数据结构保存了在它上睡眠的进程列表，例如`Linux`的等待队列。