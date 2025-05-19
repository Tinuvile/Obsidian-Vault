# Page faults

---

## Page Fault Basics

这节课的主要内容是`page fault`以及通过它实现的一系列虚拟内存功能，包括`lazy allocation`、`copy-on-write fork`、`demand paging`、`memory mapped files`。但在`xv6`中这些都没实现，采用的是直接杀掉进程的非常保守的处理方式。

我们认为虚拟内存有两个主要的优点：

- 一个是**Isolation**，隔离性。虚拟内存既提供了应用程序之间的隔离，也提供了用户空间和内核空间的隔离。

- 另一个是**Level of indirection**，提供了一层抽象，处理器和所有指令都可以使用虚拟地址，而内核会定义从虚拟地址到物理地址的映射关系。

目前的地址映射基本是静态的，即开始的时候设置好，之后基本不会再做变动。而`page fault`可以让地址映射关系变得动态起来，通过`page fault`，内核可以更新`page table`，这可以提供很多有趣的功能。

当发生`page fault`时，内核需要哪些信息才能响应它呢？

- 首先是出错的虚拟地址，即触发`page fault`的源。当出现`page fault`时，`xv6`内核会使用`trap`机制，打印出错的虚拟地址，这个地址会被保存在**STVAL**寄存器中。

- 然后是出错的原因类型。这个被存储在**SCAUSE**中，由RISC-V的文档定义。

- 还有触发`page fault`指令的程序计数器的值，这表明`page fault`在用户空间发生的位置。它存放在**SEPC**中，并且也在`trapframe->epc`中。

> 关注程序计数器值是因为在`page fault handler`中我们希望修复`page table`并重新执行对应指令。

---

## Lazy page allocation

`sbrk`系统调用是`xv6`提供给用户应用程序扩大自己的`heap`用的。当一个应用程序启动时，`sbrk`指向的是`heap`的最底端，同时也是`stack`的最顶端。这个位置通过代表进程的数据结构中的`sz`字段表示，在`proc.h`中。后面我们记作`p->sz`。

> `xv6`中栈是一个PAGESIZE大小，而堆在栈之上，可以增长。

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
  struct usyscall *usyscall; // in memlayout.h
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

当调用`sbrk`时，它的参数是`n`，代表你想申请的`page`数量，`sbrk`会扩展`heap`的上边界，内核会分配一些物理内存，并将其映射到用户应用程序的地址空间，然后将内存内容初始化为0，再返回。

> [Linux manual page](https://www.man7.org/linux/man-pages/man2/sbrk.2.html)中`sbrk`的参数是字节数。

`xv6`中，`sbrk`的实现方式是**eager allocation**，即一旦调用`sbrk`，内核会立即分配应用程序需要的物理内存。这种实现方式的坏处在于应用程序基本都倾向于申请多于需要的内存，这会导致一定程度的资源浪费。

> | **特性**    | **预先分配（Eager）**  | **惰性分配（Lazy）**        |
> | --------- | ---------------- | --------------------- |
> | **分配时机**  | 申请时立即分配物理内存      | 首次访问内存时分配物理内存（触发缺页中断） |
> | **内存利用率** | 可能浪费内存（分配后未使用的页） | 按需分配，利用率较高            |
> | **性能开销**  | 启动时开销大（需立即分配）    | 运行时延迟（处理缺页中断）         |
> | **确定性**   | 内存访问无额外延迟，适合实时系统 | 延迟不确定，可能影响实时性         |

不过基于虚拟内存和`page fault handler`，我们可以利用**lazy allocation**来解决。`sbrk`系统调用提升`p->sz`，将它增加`n`，但是内核在这个时间点并不分配任何物理内存；当应用程序使用到了新申请的那部分内存时，触发`page fault`。

这个`page fault`中，触发的虚拟地址小于当前的`p->sz`，同时大于`stack`，因此这是一个来自`heap`的地址，但内核还没有分配任何物理内存。这样的话`page fault handler`只需要通过`kalloc`函数分配一个内存`page`，初始化内容为`0`，然后将它映射到`user page table`中，最后重新执行指令即可。

> 这种方法下，从应用程序的角度看，会有一个错觉，即存在无限多可用的物理内存，内核需要解决这个问题。

修改`sys_sbrk`函数，让它只对`p->sz`加`n`，并不执行增加内存的操作。

```c
uint64
sys_sbrk(void)
{
  uint64 addr;
  int n;

  argint(0, &n);
  addr = myproc()->sz;
  myproc()->sz += n;
  // if(growproc(n) < 0)
  //   return -1;
  return addr;
}
```

然后启动`xv6`，执行`echo hi`，可以得到一个`page fault`：

```bash
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ echo hi
usertrap(): unexpected scause 0xf pid=3
            sepc=0x11ae stval=0x5008
va=20480 pte=0
panic: uvmunmap: not mapped
```

在**Shell**中执行程序时，它会先`fork`一个子进程，然后子进程通过`exec`执行`echo`。在这个过程中，它会申请一些内存，调用`sys_sbrk`，然后导致这个错误。

错误信息里，可以看到`scause`寄存器的值是15，表明它是一个`store page fault`；进程的`pid`是3，这大概是**Shell**的`pid`；`sepc`寄存器的值是`0x11ae`；最后出错的虚拟地址是`stval`的内容`0x5008`。

查看`sh.asm`汇编代码：

```asm6502
hp->s.size = nu;
    11ae:    01652423              sw    s6,8(a0)
```

可以看到这确实是一个`store`指令，另外还能注意到这个`page fault`出现在`malloc`的汇编代码中。在`malloc`中会用`sbrk`系统调用来获得一些内存，然后初始化刚刚获取的内存，在`0x11ae`位置刚刚获取的内存中写入数据，但实际上是在向未分配的内存写入数据。

然后在`usertrap`需要增加一个`scause==15`的检查并做一些特殊处理，这里只做了一个示例处理。

```c
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
  } else if (r_scause() == 15){
    uint64 va = r_stval();
    printf("usertrap(): page fault va %p\n", (void *)va);
    uint64 ka = (uint64) kalloc();
    if (ka == 0){
      p->killed = 1;
    } else {
      memset((char*)ka, 0, PGSIZE);
      if (mappages(p->pagetable, va, PGSIZE, ka, PTE_R | PTE_W | PTE_X) != 0){
        kfree((void*)ka);
        p->killed = 1;
      }
    }
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

首先打印一些调试信息，然后分配一个物理内存`page`。如果`ka`等于0，则表明没有物理内存即现在OOM（Out Of Memory）了，那我们会杀掉进程；而如果有物理内存，则首先把内存内容设置为0，然后将物理内存`page`指向用户地址空间中合适的虚拟内存地址并设置权限标志位。现在重新尝试一下：

```bash
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ echo hi
page fault 0x0000000000005008
page fault 0x0000000000014f48
panic: uvmunmap: not mapped
```

但是还是有问题，第一个`page fault`对应的虚拟地址是`0x5008`，但是在处理这个`page fault`的时候，出现了第二个`page fault`位于`0x14f48`。`uvmunmap`报错说明它尝试`unmap`的`page`不存在。

这里`unmap`的是之前`lazy allocation`但还没有用到的地址，因此这个内存并没有对应的物理地址，在`uvmunmap`中触发了：

```c
if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
```

但实际上对这个`page`我们可以不管它，直接跳到下一个即可。

```c
if((*pte & PTE_V) == 0)
      continue;
```

然后再运行，可以发现两个`page fault`后输出`hi`正常工作了。

```bash
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ echo hi
page fault 0x0000000000005008
page fault 0x0000000000014f48
hi
```

这就是一个最简单的`lazy allocation`了。

> `uvmunmap`函数在进程退出、执行新程序时都会别调用来释放空间。

但是这个实现仍然有很多可能出错，比如没有检查触发`page fault`的虚拟地址是否小于`p->sz`；另外`sys_sbrk`中的`n`是**int**型，有可能是负数，意味着缩小用户内存。这些都还需要完善。

---

## Zero Fill On Demand

下一个功能是**zero-fill-on-demand**。

一个用户程序的地址空间，有`text`区域，`data`区域，和`BBS`区域。当编译器生成二进制文件时，编译器会填入这三个区域，`text`区域存放程序的指令，`data`区域存放初始化的全局变量，`BBS`区域则包含未被初始化或初始化为0的全局和静态变量。

> 这些变量单独列出来是因为这样不用为它们分配内存

在操作系统中，如果执行`exec`，它会申请地址空间，里面存放`text`和`data`，而`BBS`里面保存了未被初始化的全局变量，这里面可能会有很多内容为0的`page`。

因此我们可以进行优化，在物理内存中只需要分配一个`page`，这个`page`的内容全是0，然后将所有虚拟地址空间中全0的`page`都映射到这一个物理页上，这样至少在程序启动时可以节省大量的物理内存分配。

不过这里映射时，我们不能允许对这个页进行写操作。当后面应用程序尝试写`BBS`中的一个`page`时，就会触发`page fault`。

对于这个`page fault`，我们需要在物理内存中申请一个新的内存页，将其内容设置为0，然后要更新这个`page`的映射关系，设置成可读可写，并将它指向新申请的物理页，最后重新执行指令。

这个优化一方面可以节省一部分内存，另一方面`exec`的工作变少了，程序可以更快启动。但相应的，`write`或其他相关的会变得更慢，因为它们都要触发`page fault`，这比`store`慢的多，`store`可能需要消耗实际访问RAM，但`page fault`需要进入内核。

---

## Copy On Write Fork

这个优化也被称为**COW fork**。

当**Shell**处理指令时，会通过`fork`创建一个子进程，`fork`会创建一个**Shell**进程的拷贝；这个子进程执行的第一件事就是调用`exec`运行一些其他程序，如`echo`。但现在`fork`创建了**Shell**地址空间的一个完整的拷贝，而`exec`做的第一件事就是丢弃这个地址空间，而用一个包含`echo`的地址空间取代它，这有些浪费。

具体来说，`xv6`的**Shell** 通常有4个`page`，调用`fork`会创建4个新的`page`，并复制父进程的到子进程中。但是调用`exec`时，我们又会释放这些`page`，并分配新的`page`来包含`echo`相关的内容。

对于这个场景一个有效的优化是：在创建子进程时共享父进程的物理内存`page`，即将子进程的PTE指向父进程对应的物理内存页。当然了，我们仍然需要保证两个进程之间的强隔离性，可以把两个进程的PTE标志位都设置成只读。

当我们需要修改内存的内容时，我们会得到`page fault`。这时需要拷贝相应的物理`page`。代码会先分配一个新的物理内存`page`，然后将`page fault`的相关物理内存页拷贝到这个里，并将它映射到子进程，设置标志位为可读写。这时父进程的标志位也可以设置为可读写了。这时再重新执行用户指令即可。

> 重新执行用户指令是调用`userret`函数。
> 
> PTE的标志位中有两位RSW，这两位保留给**supervisor software**使用，可以将它标识为当前是一个**copy-on-write page**，以给`page fault`分辨。

另外值得注意的是，这里的物理内存页可能是多对一的情况，多个用户进程指向相同的物理内存页。这时需要判断在一个进程退出时是否能立即释放相应的物理页，我们对每一个物理内存页的引用进行计数，为0时才可释放。

---

## Demand Paging

`exec`中，操作系统会加载程序内存的`text`、`data`区域，并以**eager**的方式将它们加载进`page table`。但这里我们同样可以考虑**lazy**的方式。

我们可以在虚拟地址空间中为`text`和`data`分配好地址段，但相应的PTE并不对应任何物理内存页，只需将它们的`PTE_V`设置为0即可。

应用程序会从地址0开始，这里的指令会触发第一个`page fault`。对这个情况，我们需要先从程序文件中读取数据，然后加载到物理内存中，将内存页映射到页表，再重新执行指令。

当出现OOM的情况，一个选择是撤回并释放`page`，这里最常用的策略是**Least Recently Used**。对于**dirty page**（被写过）和**non-dirty page**（只被读过），我们会选择**non-dirty page**撤回。这是因为**dirty page**必须写回磁盘，而**non-dirty page**直接修改标志位即可。PTE中有`dirty bit`和`access bit`标志位。

> 操作系统会定时扫描整个内存，将`access bit`恢复成0，这里有一些如**clock algorithm**之类的算法实现。

---

## Memory Mapped Files

这部分的思想是，将文件的一部分或者全部映射到进程的虚拟地址空间，这样就可以通过内存地址相关的`load`或者`store`指令来直接操纵文件，而无需传统的`read`、`write`系统调用。

现代操作系统会提供一个`mmap`系统调用，它接受一个虚拟内存地址`VA`、长度`len`、内存保护权限`prot`、一些标志位`flags`、一个打开文件的文件描述符`fd`和偏移量`offset`。它从文件描述符对应文件的偏移量的位置开始，映射一定长度的内容到虚拟内存地址，并加上一些保护，然后设置好PTE指向物理内存的位置。完成后有一个对应的`unmap`系统调用，这时需要将**dirty block**写回到文件中，通过PTE中的`dirty bit`识别即可。

以**lazy**方式实现的话，不会立即将文件内容拷贝到内存中，首先会记录一下PTE属于这个文件描述符，然后有一个VMA（Virtual Memory Area）结构体，这里面记录文件描述符、偏移量等信息，用来表示对应的内存虚拟地址的实际内容在哪。这样当我们得到一个位于VMA地址范围的`page fault`时，内核可以从磁盘中读取数据，并加载到内存中。

---

## 总结

这里应该内存部分的课程内容就全部结束了吧，后面还有几个Lab。虚拟内存这部分是我这门课遇到的第一个坎，这三节花了很长很长时间。

这节课中Frans说了好几次，当我们理解`page fault handler`中可以动态更新`page table`，才能理解虚拟内存有多强大。确实上到这里才意识到`page fault handler`才是整个虚拟内存系统的核心枢纽，通过它可以实现非常多高级的内存管理功能，这些功能主要在于它赋予了虚拟内存系统非常高的灵活性，可以随时按照需求调整，同时还能保存安全与隔离。
