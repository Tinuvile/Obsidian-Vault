# Traps and system calls

## 陷阱与系统调用

---

有三种情况会导致CPU暂停普通指令的执行并强行将控制权转移给处理该事件的特殊代码：

- 第一种情况是系统调用，用户程序执行`ecall`指令请求内核为其执行某些操作时；

- 第二种情况是异常，当指令执行了非法操作时；

- 第三种情况是设备中断，当设备发出信号提示其需要注意时。

我们将这些情况统称为`trap`。通常的流程是：`trap`强制将控制权转移到内核`->`内核保存寄存器和其他状态以便后续恢复执行`->`内核执行适当的处理程序代码`->`内核恢复保存的状态并从`trap`返回`->`原始代码从中断处继续执行。

`xv6`内核负责统一处理这三种情况。这对于**System call**是自然的；对于**Interrupt**也一样，因为隔离要求用户进程不能直接使用设备，而只有内核拥有处理设备所需权限；对**Exception**来说，`xv6`通过杀死违规程序来响应来自用户空间的所有异常。

`xv6`的陷阱处理分为四个阶段：RISC-V CPU执行的硬件操作、为内核代码做准备的汇编向量、C陷阱处理程序、系统调用或设备驱动服务例程。

---

## RISC-V trap machinery

## RISC-V陷阱机制

每个RISC-V寄存器都有一组控制寄存器，内核通过写入这些寄存器来告诉CPU如何处理陷阱，且内核可以通过阅读寄存器来得知已经发生的`trap`。其中几个最重要的寄存器是：

- `stvec`：内核将其陷阱处理程序的地址写入这里，RISC-V通过跳转到这里来处理陷阱；

- `sepc`：RISC-V将发生陷阱时的程序计数器保存在这里（随后`pc`会被`stvec`覆盖），`sret`指令（从陷阱返回）将`sepc`重新复制到`pc`，内核可以写入`sepc`以控制`sret`的去向；

- `scause`：RISC-V在这里放置一个数字用于描述陷阱的原因；

- `sscratch`：用于陷阱处理时安全保存用户态的上下文信息，这是监督者模式的临时寄存器（Supervisor Scratch Register）；

- `sstatus`：其中的**SIE**位控制设备中断是否启用。如果内核清除SIE，RISC-V会推迟设备中断直到内核设置SIE。**SPP**位指示陷阱是来自用户模式还是监督模式，并控制`sret`返回到哪种模式。

上述寄存器都无法在用户模式下读写，它们用于监督者模式下的陷阱处理。对于机器模式下的陷阱处理，`xv6`有一组等效的控制寄存器，它们仅在定时器中断的特殊情况下被启用。多核芯片上的每个CPU都有自己的一组控制寄存器。

当需要强制进入`trap`时，RISC-V硬件会对所有`trap`类型（除了定时器中断）执行以下操作：

1. 如果陷阱是设备中断且`sstatus`的SIE位被清除，则不用执行以下操作；

2. 清除SIE位来禁用中断；

3. 将`pc`复制到`sepc`；

4. 将目前的模式（`user`或者`supervisor`）保存到`sstatus`寄存器中的SPP位；

5. 设置`scause`反映陷阱原因；

6. 将当前模式设置为`supervisor`；

7. 将`stvec`复制到`pc`；

8. 从新的`pc`开始执行。

值得注意的是，CPU不会切换到内核页表、内核栈，也不会保存除了`pc`外的其他寄存器。

---

## Traps from user space

## 来自用户空间的陷阱

当用户程序进行系统调用（`ecall`指令）、执行非法操作或设备中断时都可能发生`trap`，处理用户空间陷阱的代码流程如下：

1. `uservec`（在`trampoline.S`中）
   
   这是陷阱处理的入口，由`stvec`寄存器指向这里。负责保存用户态寄存器到**TRAPFRAME**并且切换内核页表和内核栈，然后跳转到`usertrap`。
   
   ```asm6502
   .globl uservec
   uservec:    
       #
           # trap.c sets stvec to point here, so
           # traps from user space start here,
           # in supervisor mode, but with a
           # user page table.
           #
   
           # save user a0 in sscratch so
           # a0 can be used to get at TRAPFRAME.
           csrw sscratch, a0
   
           # each process has a separate p->trapframe memory area,
           # but it's mapped to the same virtual address
           # (TRAPFRAME) in every process's user page table.
           li a0, TRAPFRAME
   
           # save the user registers in TRAPFRAME
           sd ra, 40(a0)
           sd sp, 48(a0)
           sd gp, 56(a0)
           sd tp, 64(a0)
           sd t0, 72(a0)
           sd t1, 80(a0)
           sd t2, 88(a0)
           sd s0, 96(a0)
           sd s1, 104(a0)
           sd a1, 120(a0)
           sd a2, 128(a0)
           sd a3, 136(a0)
           sd a4, 144(a0)
           sd a5, 152(a0)
           sd a6, 160(a0)
           sd a7, 168(a0)
           sd s2, 176(a0)
           sd s3, 184(a0)
           sd s4, 192(a0)
           sd s5, 200(a0)
           sd s6, 208(a0)
           sd s7, 216(a0)
           sd s8, 224(a0)
           sd s9, 232(a0)
           sd s10, 240(a0)
           sd s11, 248(a0)
           sd t3, 256(a0)
           sd t4, 264(a0)
           sd t5, 272(a0)
           sd t6, 280(a0)
   
       # save the user a0 in p->trapframe->a0
           csrr t0, sscratch
           sd t0, 112(a0)
   
           # initialize kernel stack pointer, from p->trapframe->kernel_sp
           ld sp, 8(a0)
   
           # make tp hold the current hartid, from p->trapframe->kernel_hartid
           ld tp, 32(a0)
   
           # load the address of usertrap(), from p->trapframe->kernel_trap
           ld t0, 16(a0)
   
           # fetch the kernel page table address, from p->trapframe->kernel_satp.
           ld t1, 0(a0)
   
           # wait for any previous memory operations to complete, so that
           # they use the user page table.
           sfence.vma zero, zero
   
           # install the kernel page table.
           csrw satp, t1
   
           # flush now-stale user entries from the TLB.
           sfence.vma zero, zero
   
           # jump to usertrap(), which does not return
           jr t0
   ```
   
   RISC-V硬件在发生`trap`时不切换页表，在跳转向`stvec`指向的指令时使用的仍然是用户页表，因此用户页表必须包含对`uservec`的映射。然后`uservec`切换`satp`以指向内核页表。为了在切换后继续执行指令，`uservec`必须在内核页表和用户页表中保持相同的地址映射。
   
   `xv6`通过一个包含`uservec`的`trampoline`页来满足这些约束。`xv6`在内核页表和用户页表中将`trampoline`页面映射到相同的虚拟地址，这个虚拟地址就是**TRAMPOLINE**。`trampoline`的内容在`trampoline.S`中设置，并且`stvec`被设置为`uservec`。
   
   当`trap`发生时，CPU进入监督者模式，所有32个通用寄存器保持用户态的值。然后代码运行`csrrw sscratch, a0`，交换`a0`和`sscratch`寄存器的内容，这样`uservec`通过内核预先设置在`sscratch`中的值（**TRAPFRAME**的虚拟地址）获得了一个可用寄存器，`a0`也获得了TRAPFRAME的虚拟地址。
   
   ```tex
   操作前:
   a0 = 用户数据
   sscratch = TRAPFRAME地址 (由内核预设)
   
   操作后:
   a0 = TRAPFRAME地址
   sscratch = 用户原始a0值
   ```
   
   > 这里书中使用的是`csrrw`交换指令，但是项目代码中使用的是`csrw sscratch, a0`和`li a0, TRAPFRAME`，首先单向将`a0`的值写入`sscratch`，然后再加载TRAPFRAME的虚拟地址到`a0`。
   
   然后`uservec`需要保存用户寄存器。在进程创建时，进入用户空间前，内核会将`sscratch`设置为指向每个进程的`trapframe`，该`trapframe`有空间保存所有用户寄存器。而此时`satp`仍然在使用用户页表，因此`uservec`需要将`trapframe`映射在用户地址空间中。在创建每个进程时，`xv6`为进程的`trapframe`分配一个页，并安排它始终映射在用户虚拟地址**TRAPFRAME**处，该地址位于**TRAMPOLINE**下方。进程的`p->trapframe`也指向`trapframe`，供内核通过内核页表使用。即
   
   ```tex
   // 用户态通过VA访问：
   TRAPFRAME->a0 = x;  // 使用用户页表
   
   // 内核态通过PA访问：
   p->trapframe->kernel_sp = ...; // 使用内核页表
   ```
   
   `trapframe`包含指向当前进程内核栈的指针、当前CPU的`hartid`、`usertrap`的地址以及内核页表的地址等。
   
   ```c
   struct trapframe {
       /*  0 */ uint64 kernel_satp;  // 内核页表地址
       /*  8 */ uint64 kernel_sp;    // 内核栈指针
       /* 16 */ uint64 kernel_trap;  // usertrap函数地址
       /* 32 */ uint64 kernel_hartid;// CPU核心ID
       ...
   };
   ```
   
   `uservec`检索这些值，将`satp`切换到内核页表，见`csrw satp, t1`一句，并调用`usertrap`。

2. `usertrap`（在`trap.c`中）
   
   这是陷阱处理的核心逻辑，负责判断陷阱类型并执行对应的处理程序。
   
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
   
   它首先更改`stvec`，以便内核陷阱由`kernelvec`处理，`w_stvec((uint64)kernelvec)`设置内核陷阱向量，这样可以防止在内核态处理用户陷阱时出现陷阱递归；然后保存`sepc`寄存器，`p->trapframe->epc = r_sepc()`，防止进程切换覆盖；如果陷阱类型是系统调用，由`syscall`处理，设备中断由`devintr`处理，异常直接杀死进程。
   
   `p->trapframe->epc += 4`，系统调用路径下，会将用户`pc`加4，指向`ecall`的下一条指令。退出时，检查进程是否被杀死或是否需要让出CPU。定时器中断（`which_dev == 2`）会触发调度，防止进程独占CPU。

3. `usertrapret`（在`trap.c`中）
   
   负责重新设置`sstatus`和`sepc`寄存器，并将`stvec`重新指向`uservec`。
   
   ```c
   //
   // return to user space
   //
   void
   usertrapret(void)
   {
     struct proc *p = myproc();
   
     // we're about to switch the destination of traps from
     // kerneltrap() to usertrap(), so turn off interrupts until
     // we're back in user space, where usertrap() is correct.
     intr_off();
   
     // send syscalls, interrupts, and exceptions to uservec in trampoline.S
     uint64 trampoline_uservec = TRAMPOLINE + (uservec - trampoline);
     w_stvec(trampoline_uservec);
   
     // set up trapframe values that uservec will need when
     // the process next traps into the kernel.
     p->trapframe->kernel_satp = r_satp();         // kernel page table
     p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
     p->trapframe->kernel_trap = (uint64)usertrap;
     p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()
   
     // set up the registers that trampoline.S's sret will use
     // to get to user space.
   
     // set S Previous Privilege mode to User.
     unsigned long x = r_sstatus();
     x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
     x |= SSTATUS_SPIE; // enable interrupts in user mode
     w_sstatus(x);
   
     // set S Exception Program Counter to the saved user pc.
     w_sepc(p->trapframe->epc);
   
     // tell trampoline.S the user page table to switch to.
     uint64 satp = MAKE_SATP(p->pagetable);
   
     // jump to userret in trampoline.S at the top of memory, which 
     // switches to the user page table, restores user registers,
     // and switches to user mode with sret.
     uint64 trampoline_userret = TRAMPOLINE + (userret - trampoline);
     ((void (*)(uint64))trampoline_userret)(satp);
   }
   ```
   
   它设置RISC-V控制寄存器，将`stvec`指向`uservec`，填充`trapframe`字段，并将`sepc`设置为之前保存的用户程序计数器。最后在用户和内核页表中都有映射的`trampoline`页面上调用`userret`，因为`userret`将切换页表。
   
   ```tex
   虚拟地址空间布局：
   用户空间：
   0x3ffffffe000 (TRAMPOLINE) → 物理页A（包含trampoline代码）
   内核空间：
   0x3ffffffe000 (TRAMPOLINE) → 相同物理页A
   ```

4. `userret`（在`trampoline.S`中）
   
   负责切换回用户页表并恢复用户态寄存器。
   
   ```asm6502
   .globl userret
   userret:
           # userret(pagetable)
           # called by usertrapret() in trap.c to
           # switch from kernel to user.
           # a0: user page table, for satp.
   
           # switch to the user page table.
           sfence.vma zero, zero
           csrw satp, a0
           sfence.vma zero, zero
   
           li a0, TRAPFRAME
   
           # restore all but a0 from TRAPFRAME
           ld ra, 40(a0)
           ld sp, 48(a0)
           ld gp, 56(a0)
           ld tp, 64(a0)
           ld t0, 72(a0)
           ld t1, 80(a0)
           ld t2, 88(a0)
           ld s0, 96(a0)
           ld s1, 104(a0)
           ld a1, 120(a0)
           ld a2, 128(a0)
           ld a3, 136(a0)
           ld a4, 144(a0)
           ld a5, 152(a0)
           ld a6, 160(a0)
           ld a7, 168(a0)
           ld s2, 176(a0)
           ld s3, 184(a0)
           ld s4, 192(a0)
           ld s5, 200(a0)
           ld s6, 208(a0)
           ld s7, 216(a0)
           ld s8, 224(a0)
           ld s9, 232(a0)
           ld s10, 240(a0)
           ld s11, 248(a0)
           ld t3, 256(a0)
           ld t4, 264(a0)
           ld t5, 272(a0)
           ld t6, 280(a0)
   
       # restore user a0
           ld a0, 112(a0)
   
           # return to user mode and user pc.
           # usertrapret() set up sstatus and sepc.
           sret
   ```
   
   它通过`a0`接收进程的用户页表指针，然后将`satp`切换到进程的用户页表，并在切换页表后加载**TRAPFRAME**地址到`a0`，然后恢复所有寄存器，并将`trapframe`中保存的用户 `a0`复制到`sscratch`。接下来，`userret`从`trapframe`中恢复保存的用户寄存器，最后交换`a0`和`sscratch`以恢复用户`a0`并为下一次`trap`保存**TRAPFRAME**，然后使用`sret`回到用户空间。
   
   > `a0`寄存器需要最后恢复，因为前面指令都依赖`a0`作为**TRAPFRAME**基址。
   > 
   > 另外，上述描述与代码有一些出入，目前代码的流程应该是：恢复所有寄存器，最后恢复`a0`，然后直接返回用户空间。这是因为内核在进程调度时会重新初始化陷阱帧。

---

## Code：Calling system calls

## 代码：调用系统调用

这部分介绍用户调用如何到达内核中`exec`系统调用。

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();

    if(p->trace_mask & (1 << num)) {
      printf("%d: syscall %s -> %ld\n",
        p->pid,
        syscall_names[num],
        p->trapframe->a0
      );
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

用户代码将`exec`的参数放入寄存器`a0`和`a1`中，并将系统调用数放入`a7`，系统调用数会用于与`syscalls`数组中的条目匹配，这是一个函数表来索引`syscalls`。然后`ecall`触发`trap`进入内核，执行系统调用触发的相应步骤。

当系统调用实现函数返回时，`syscall`记录返回值到`p->trapframe->a0`，这将导致原始用户空间调用的`exec`返回该值。因为在RISC-V的C调用约定中，返回值放在`a0`中。

系统调用通常返回负数表示错误，返回零或正数表示成功。如果系统调用数无效，`syscall`会打印错误并返回`-1`。

---

## Code：System call arguments

## 代码：系统调用参数

内核中的系统调用实现需要找到用户代码传递的参数。参数最初位于寄存器中，内核陷阱代码将所有用户寄存器保存到了当前进程的陷阱帧（`trapframe`）中。`argint`、`argaddr`和`argfd`函数分别从陷阱帧中检索第`n`个系统调用参数，分别作为整数、指针和文件描述符。它们都调用`argraw`来检索适当的已保存的用户寄存器。

当系统调用使用指针作为参数传递时，一方面可能用户程序存在错误或恶意，另外`xv6`内核的页表映射与用户页表映射也不同，因此内核无法使用普通指令从用户提供的地址加载或存储数据。

内核实现了安全地在用户提供的地址之间传输数据的功能。例如`fetchstr`，文件系统调用使用它从用户空间检索字符串文件名参数，它又调用`copyinstr`。

```c
// Fetch the nul-terminated string at addr from the current process.
// Returns length of string, not including nul, or -1 for error.
int
fetchstr(uint64 addr, char *buf, int max)
{
  struct proc *p = myproc();
  if(copyinstr(p->pagetable, buf, addr, max) < 0)
    return -1;
  return strlen(buf);
}
```

`copyinstr`从用户页表的虚拟地址`srcva`中复制最多`max`字节到`dst`。它使用`walkaddr`遍历页表，以确定`srcva`的物理地址`pa0`。由于内核将所有物理RAM地址映射到相同的内核虚拟地址，`copyinstr`可以直接将字符串字节从`pa0`复制到`dst`。`walkaddr`会检查用户提供的虚拟地址是否属于进程的用户地址空间。

```c
// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  uint64 n, va0, pa0;
  int got_null = 0;

  while(got_null == 0 && max > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > max)
      n = max;

    char *p = (char *) (pa0 + (srcva - va0));
    while(n > 0){
      if(*p == '\0'){
        *dst = '\0';
        got_null = 1;
        break;
      } else {
        *dst = *p;
      }
      --n;
      --max;
      p++;
      dst++;
    }

    srcva = va0 + PGSIZE;
  }
  if(got_null){
    return 0;
  } else {
    return -1;
  }
}
```

---

## Traps from kernel space

## 内核空间的陷阱

`xv6`根据用户代码和内核代码的执行情况，对CPU的陷阱寄存器进行了不同的配置。当内核在CPU上执行时，内核将`stvec`指向`kernelvec`处的汇编代码。由于`xv6`已经处于内核态，`kernelvec`可以通过`satp`被设置为内核页表，并且栈指针指向一个有效的内核栈。`kernelvec`会保存所有寄存器，以便被中断的代码后面能不受干扰的恢复执行。

> `kernelvec`将寄存器保存在被中断的内核线程的栈上，因为陷阱可能导致切换到另一个线程，然后在新线程的栈上返回，而被中断线程的寄存器可以安全地保存在其栈上。

`kernelvec`在保存寄存器后跳转到`kerneltrap`，`kerneltrap`调用`devintr`来检查和处理设备中断，调用`panic`来处理异常。

```c
// interrupts and exceptions from kernel code go here via kernelvec,
// on whatever the current kernel stack is.
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

如果`kerneltrap`由于定时器中断而被调用，并且一个非调度器线程的进程的内核线程正在运行，那么`kerneltrap`会调用`yield`来给其他线程运行的机会。

---

## Page-fault exceptions

## 缺页异常

`xv6`对异常的处理很简单：

- 如果在用户空间，内核会终止出错的进程；

- 如果在内核空间，内核会崩溃。

很多内核使用缺页异常（page-fault）来实现`copy-on-write`（COW）`fork`。父进程和子进程可以通过COW安全地共享物理内存。当CPU无法将虚拟地址转为物理地址时，CPU会生成一个缺页异常错误。

RISC-V有三种不同类型的`page fault`：

- `load page fault`加载页面错误（当加载指令无法转换其虚拟地址时）；

- `store page fault`存储页面错误（当存储指令无法转换其虚拟地址时）；

- `instruction page fault`指令页面错误（当指令的地址无法转换时）

`scause`寄存器中的值指示页面错误的类型，`stval`寄存器包含无法转换的地址。

`COW fork`让父进程和子进程共享所有物理页面，但将它们映射为只读。因此，当子进程或父进程执行存储指令时，RISC-V CPU会引发缺页异常。然后内核做出响应，它复制包含错误地址的页面，然后将一个副本映射给子进程地址空间读写，另一个副本给父进程。更新页表后，内核在引发错误的指令处恢复故障进程。此时内核已经更新`PTE`运行写入，因此现在指令可以正确执行。

另一个利用页面错误的功能是从磁盘分页。如果应用程序需要的内存超过了可用的物理RAM，内核可用`evict`一些页，将它们写入磁盘等存储设备，并将它们的`PTE`标记为无效。如果应用程序读写这些页面就会触发`page fault`，然后内核检查错误地址，如果该地址属于磁盘上的页面，内核就会分配一页物理内存，将该页面从磁盘读到该内存中，然后更新`PTE`，恢复应用程序。
