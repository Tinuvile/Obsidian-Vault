# traps源码阅读

---

这部分的文件主要是`trampoline.S`、`kernelvec.S`和`trap.c`。

---

## trampoline.S

### uservec

这是一个汇编代码文件，在`xv6`中负责处理用户态和内核态的切换。

```nasm
.section trampsec  // 声明代码位于trampsec段
.globl trampoline  // 声明trampoline代码起始地址
.globl usertrap    // 声明trap处理函数
trampoline:        // trampoline代码起始标签
.align 4           // 对齐
```

首先是一些声明。

```nasm
.globl uservec     // 声明用户态trap入口
```

然后是`uservec`代码，它在用户态到内核态切换时保存寄存器状态并设置内核环境。`trap.c`会设置**STVEC**寄存器指向这里，因此用户空间中的陷阱触发后从这里开始，使用用户页表，但是在`supervisor`模式下。

```nasm
csrw sscratch, a0  // 将用户a0寄存器值暂存到sscratch寄存器
li a0, TRAPFRAME   // 加载TRAPFRAME虚拟地址到a0

// 下面这些里面只少了112，因为它是专门用来存原始a0值
sd ..., ...(a0)    // 保存用户寄存器到trapframe

csrr t0, sscratch  // 恢复用户原始a0值
sd t0, 112(a0)     // 存入p->trapframe->a0

ld sp, 8(a0)       // 加载内核栈指针，从p->trapframe->kernel_sp
ld tp, 32(a0)      // 设置当前hartid，从p->trapframe->kernel_hartid
ld t0, 16(a0)      // 加载usertrap函数地址，从trapframe->kernel_trap
ld t1, 0(a0)       // 获取内核页表地址，从trapframe->kernel_satp
```

`a0`前面的数字是`trapframe`结构体中各字段的内存偏移量，`trapframe`的定义在[`proc.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h#L43)中。

然后`sfence.vma zero, zero`是内存屏障指令，会保证之前的内存操作完成。

```nasm
csrw satp, t1      // 切换内核页表
```

这时`sfence.vma zero zero`会刷新TLB，清除旧页表缓存。

然后`jr t0`跳转到`usertrap`函数。

### userret

由`trap.c`中的`usertrapret`调用，负责从内核态回到用户态。

```nasm
sfence.vma zero, zero
csrw satp, a0
sfence.vma zero, zero
```

先切换回用户页表。

> 这里的`a0`是内核`a0`而非前面的用户`a0`，用户`a0`在先前被修改，但这里的内核`a0`是内核准备的参数值。

然后加载TRAPFRAME虚拟地址，保存寄存器等。

```nasm
li a0, TRAPFRAME
```

最后会把重新存回用户`a0`寄存器的值，并返回用户空间。

```nasm
ld a0, 112(a0)
sret
```

---

## kernelvec.S

`kernelvec`是内核陷阱的入口，处理来自内核代码中的中断或异常。

```nasm
addi sp, sp, -256
```

它先分配256字节栈空间来保存寄存器的值。然后通过`call kerneltrap`调用C处理函数，处理完成后再恢复寄存器状态。

---

## trap.c

### trapinithart

`trapinithart`是内核陷阱处理的核心初始化函数。

```c
// set up to take exceptions and traps while in the kernel.
void
trapinithart(void)
{
  w_stvec((uint64)kernelvec);
}
```

它通过`w_stvec`设置陷阱向量基址寄存器，将内核陷阱处理程序入口地址设置为`kernelvec`。

### kerneltrap

这是处理内核空间陷阱的核心函数。

```c
void 
kerneltrap()
{
  int which_dev = 0;
  uint64 sepc = r_sepc();
  uint64 sstatus = r_sstatus();
  uint64 scause = r_scause();
  
  if((sstatus & SSTATUS_SPP) == 0)
    panic("kerneltrap: not from supervisor mode");
  if(intr_get() != 0)
    panic("kerneltrap: interrupts enabled");

  if((which_dev = devintr()) == 0){
    // interrupt or trap from an unknown source
    printf("scause=0x%lx sepc=0x%lx stval=0x%lx\n", scause, r_sepc(), r_stval());
    panic("kerneltrap");
  }

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2 && myproc() != 0)
    yield();

  // the yield() may have caused some traps to occur,
  // so restore trap registers for use by kernelvec.S's sepc instruction.
  w_sepc(sepc);
  w_sstatus(sstatus);
}
```

首先保存关键寄存器的值，然后验证是否来自`supervisor mode`并检查中断是否已经禁用，再尝试用`devintr`处理中断。若是定时器中断且进程存在，则调用`yield`触发进程调度。最后恢复寄存器的值。

### usertrap

这是处理用户空间陷阱的核心函数。

```c
if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");
```

首先检查确认来自用户模式，然后设置入口地址，后续若触发中断将交由内核陷阱处理程序来处理。

```c
w_stvec((uint64)kernelvec);
```

然后保存用户程序计数器：

```c
struct proc *p = myproc();
  
// save user program counter.
p->trapframe->epc = r_sepc();
```

再根据`scause`寄存器的值来确定不同的处理方法：

```c
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
    printf("page fault %p\n", (void *)va);
    uint64 ka = (uint64) kalloc();
    if (ka == 0){
      p->killed = 1;
    } else {
      memset((char*)ka, 0, PGSIZE);
      va = PGROUNDDOWN(va);
      if (mappages(p->pagetable, va, PGSIZE, ka, PTE_W | PTE_U | PTE_R) != 0){
        kfree((void*)ka);
        p->killed = 1;
      }
    }
  } else {
    printf("usertrap(): unexpected scause 0x%lx pid=%d\n", r_scause(), p->pid);
    printf("            sepc=0x%lx stval=0x%lx\n", r_sepc(), r_stval());
    setkilled(p);
  }
```

当定时器中断则触发`yield`让出CPU：

```c
if(which_dev == 2)
    yield();
```

最后调用`usertrapret`返回用户空间。

### usertrapret

这个函数用于返回用户空间。

首先`intr_off`关闭中断，然后重新设置用户陷阱处理入口：

```c
uint64 trampoline_uservec = TRAMPOLINE + (uservec - trampoline);
w_stvec(trampoline_uservec);
```

并重新配置`trapframe`和`sstatus`寄存器：

```c
  p->trapframe->kernel_satp = r_satp();         // kernel page table
  p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
  p->trapframe->kernel_trap = (uint64)usertrap;
  p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()
  
  unsigned long x = r_sstatus();
  x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
  x |= SSTATUS_SPIE; // enable interrupts in user mode
  w_sstatus(x);
```

返回地址使用之前保存的：

```c
w_sepc(p->trapframe->epc);
```

然后切换页表，重新跳转到用户空间。

```c
uint64 satp = MAKE_SATP(p->pagetable);
uint64 trampoline_userret = TRAMPOLINE + (userret - trampoline);
((void (*)(uint64))trampoline_userret)(satp);  // 执行userret汇编代码
```

### 其他

`devintr`和`clockintr`涉及中断，后面再补充。






























