# Interrupts

---

## 真实操作系统内存使用情况

运行`top`指令，在`Mem`这一行，大部分被用掉的内存，都是被`buff/cache`用了。

![](..\image\2025-04-27-19-17-14-image.png)

大部分操作系统运行时几乎没有任何空闲的内存。因此很多时候要分配内存都要先撤回一些内存。

`VIRT`表示的是虚拟内存地址空间的大小，`RES`是实际使用的内存数量。实际使用的内存数量远小于地址空间的大小。之前介绍的基于虚拟内存和`page fault`提供的功能在这都有所使用。

---

## Interrupt硬件部分

中断对应的场景是：硬件需要得到操作系统的关注。操作系统会保存当前的工作，处理中断，再恢复先前的工作。与系统调用、`page fault`的过程非常相似，因此使用相同的机制。

中断和系统调用的差别主要有三点：

- **asynchronous**：当硬件生成中断时，Interrupt handler 与当前运行的进程在 CPU 上没有任何关联；但系统调用发生在运行进程的 context 下。

- **concurrency**：对中断来说，CPU 和生成中断的设备是并行运行的。

- **program device**：每个设备都需要被编程，如网卡、UART。

我们在这里主要关注外部设备的中断，而非定时器中断或软件中断。主板上，各种线路将外设与 CPU 连接在一起，处理器通过PLIC（Platform Level Interrupt Control）来处理设备中断。

![plic_cpu.png](F:\MIT6.S081-xv6-labs-2024\note\image\plic_cpu.png)

共有53个不同的来自设备的中断。中断到达 PLIC 后，PLIC 会路由这些中断，图的右下角是 CPU 的核，PLIC 会将中断路由到某一个 CPU 的核。如果所有 CPU 核都在处理中断，PLIC会保留中断直到有一个 CPU 核可以用来处理中断。因此 PLIC 需要保存一些数据来跟踪中断状态。

---

## 设备驱动概述

管理设备的代码称为驱动，所有驱动都在内核中。大部分驱动的代码都分为两个部分，`bottom/top`。这里以 UART 设备的驱动为例。

`bottom`部分通常是`Interrupt handler`，当中断送到 CPU 并被接收，CPU 会调用相应的`Interrupt handler`，它并不运行在任何特定进程的`context`中，它只处理中断。但也因此它存在一些限制，因为进程的`page table`并不知道应该从哪个地址读写数据，也就无法直接从`Interrtupt handler`读写数据。

`top`部分是用户进程或内核其他部分调用的接口。对 UART 是`read/write`接口。这部分通常与用户的进程交互并进行数据读写。

通常，驱动中会有一些队列，或者说 buffer。`top`和`bottom`部分的代码都会从队列中读写数据。这里的队列可以将并行运行的设备和 CPU 解耦开。

对设备的编程通常是通过**memory mapped I/O**完成的。设备地址出现在物理地址的特定区间，这由主板制造商决定。操作系统知道这些设备位于物理地址空间的具体位置，然后通过`load/store`指令对这些地址进行编程，实际就是读写设备的控制寄存器来操作设备实现相应的行为。

![](C:\Users\ASUS\AppData\Roaming\marktext\images\2025-04-27-20-15-13-image.png)

16550 是 QEMU 模拟的 UART 设备，QEMU 用这个模拟的设备与键盘和 Console 进行交互。图中表明了芯片拥有的寄存器。例如控制寄存器`000`，写它可以将数据写入寄存器中，读它可以读出寄存器中的内容。UART 可以通过串口发送数据 bit，线路另一侧的 UART 芯片可以将数据 bit 重新组合成字节。以及控制寄存器`001`，可以通过它控制 UART是否产生中断。

> 当通过`load`将数据写入`Transmit Holding Register`，UART 芯片会通过串口线将这个 Byte 送出，完成发送后 UART 会生成一个中断给内核，这时才能写入下一个数据。上图的 UART 芯片有一个容量16的 FIFO。

---

## 在`xv6`中设置中断

我们讨论`$ls`。

`xv6`中，**Shell**会输出提示符`$`。实际过程是：设备会将字符传输给 UART 的寄存器，UART 在发送完字符后产生一个中断，在 QEMU 中，模拟的线路的另一端会有另一个模拟的 UART 芯片，这个芯片连接到虚拟的 Console，它会将`$`显示在 console 上。

`ls`是用户输入的字符。键盘连接到 UART 的输入线路，当键盘上一个按键被按下，UART 芯片会将按键字符通过串口线发送到另一端的 UART 芯片。它会先将数据 bit 合并成一个 Byte，然后再产生一个中断，并告诉处理器有一个来自键盘的字符，之后`Interrupt handler`会处理来自 UART 的字符。

> RISC-V 有许多与中断有关的寄存器：
> 
> - **SIE**寄存器：这个寄存器中有一个 bit (E) 专门针对外部设备的中断；另有一个 bit (S) 专门针对软件中断和一个 bit (T) 针对定时器中断。软件中断可能由一个 CPU核触发给另一个 CPU 核。
> 
> - **SSTATUS**寄存器：这个寄存器中有一个 bit 来打开或关闭中断。
> 
> 每个CPU核都有独立的 SIE 和 SSTATUS 寄存器。SIE 单独控制特定的中断，SSTATUS 控制所有的中断。
> 
> - **SIP**寄存器：处理器通过这个寄存器查看是什么类型的中断。
> 
> - **SCAUSE**寄存器：表明当前状态的原因是中断。
> 
> - **STVEC**寄存器：保存当`trap`，`page fault`或中断发生时，CPU 运行的用户程序的程序计数器。

`start`函数先将所有的中断都设置在`Supervisor mode`，然后设置 SIE 寄存器来接收外部设备、软件和定时器中断，之后初始化定时器。

```c
void
start()
{
  // set M Previous Privilege mode to Supervisor, for mret.
  unsigned long x = r_mstatus();
  x &= ~MSTATUS_MPP_MASK;
  x |= MSTATUS_MPP_S;
  w_mstatus(x);

  // set M Exception Program Counter to main, for mret.
  // requires gcc -mcmodel=medany
  w_mepc((uint64)main);

  // disable paging for now.
  w_satp(0);

  // delegate all interrupts and exceptions to supervisor mode.
  w_medeleg(0xffff);
  w_mideleg(0xffff);
  w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);

  // configure Physical Memory Protection to give supervisor mode
  // access to all of physical memory.
  w_pmpaddr0(0x3fffffffffffffull);
  w_pmpcfg0(0xf);

  // ask for clock interrupts.
  timerinit();

  // keep each CPU's hartid in its tp register, for cpuid().
  int id = r_mhartid();
  w_tp(id);

  // switch to supervisor mode and jump to main().
  asm volatile("mret");
}
```

`main`函数中，`console`是第一个外设。

```c
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    ...
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    ...
  }
  scheduler();
}
```

在`consoleinit`中，先初始化锁，然后调用`uartinit`，这个函数会配置好 UART 芯片以供使用。

```c
void
consoleinit(void)
{
  initlock(&cons.lock, "cons");

  uartinit();

  // connect read and write system calls
  // to consoleread and consolewrite.
  devsw[CONSOLE].read = consoleread;
  devsw[CONSOLE].write = consolewrite;
}
```

`uartinit`的流程是先关闭中断，然后设置波特率（串口线的传输速率），设置字符长度为8 bit，并重置 FIFO，最后打开中断。

```c
void
uartinit(void)
{
  // disable interrupts.
  WriteReg(IER, 0x00);

  // special mode to set baud rate.
  WriteReg(LCR, LCR_BAUD_LATCH);

  // LSB for baud rate of 38.4K.
  WriteReg(0, 0x03);

  // MSB for baud rate of 38.4K.
  WriteReg(1, 0x00);

  // leave set-baud mode,
  // and set word length to 8 bits, no parity.
  WriteReg(LCR, LCR_EIGHT_BITS);

  // reset and enable FIFOs.
  WriteReg(FCR, FCR_FIFO_ENABLE | FCR_FIFO_CLEAR);

  // enable transmit and receive interrupts.
  WriteReg(IER, IER_TX_ENABLE | IER_RX_ENABLE);

  initlock(&uart_tx_lock, "uart");
}
```

运行完`uartinit`后，UART 就可以生成中断了。但还没有对 PLIC 编程，因此中断不能被 CPU 感知。在`main`函数中还要调用`plicinit`函数。

```c
void
plicinit(void)
{
  // set desired IRQ priorities non-zero (otherwise disabled).
  *(uint32*)(PLIC + UART0_IRQ*4) = 1;
  *(uint32*)(PLIC + VIRTIO0_IRQ*4) = 1;
}
```

PLIC 与外设一样，也占用了一个 I/O 地址。这里设置了 PLIC 会接收哪些中断，进而将中断路由到 CPU。在上面设置了 UART 和 IO磁盘的中断。

```c
void
plicinithart(void)
{
  int hart = cpuid();

  // set enable bits for this hart's S-mode
  // for the uart and virtio disk.
  *(uint32*)PLIC_SENABLE(hart) = (1 << UART0_IRQ) | (1 << VIRTIO0_IRQ);

  // set this hart's S-mode priority threshold to 0.
  *(uint32*)PLIC_SPRIORITY(hart) = 0;
}
```

`plicinit`由0号 CPU 运行。之后，每个 CPU 的核都需要调用`plicinithart`函数表明可以处理哪些外设中断。上面每个 CPU 核都可以处理来自 UART 和 VIRTIO 的中断，中断的优先级被设置为0。

目前，生成中断的外部设备和传递中断给 CPU 的 PLIC 都设置好了，但 CPU 还没有设置好接收中断，需要设置 SSTATUS 寄存器。在`main`的最后调用了`scheduler`函数。

```c
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

`scheduler`函数主要是运行进程。但在实际运行进程前会执行`intr_on`函数设置 SSTATUS 寄存器来使 CPU 能接收中断。

```c
static inline void
intr_on()
{
  w_sstatus(r_sstatus() | SSTATUS_SIE);
}
```

以上就是中断的基本配置。

---

## UART驱动的top部分

接下来介绍如何从 Shell 程序输出提示符`$`到 Console。首先是`init.c`中的`main`函数，这是系统启动后运行的第一个进程。

```c
int
main(void)
{
  int pid, wpid;

  if(open("console", O_RDWR) < 0){
    mknod("console", CONSOLE, 0);
    open("console", O_RDWR);
  }
  dup(0);  // stdout
  dup(0);  // stderr

  for(;;){
    printf("init: starting sh\n");
    pid = fork();
    if(pid < 0){
      printf("init: fork failed\n");
      exit(1);
    }
    if(pid == 0){
      exec("sh", argv);
      printf("init: exec sh failed\n");
      exit(1);
    }

    for(;;){
      // this call to wait() returns if the shell exits,
      // or if a parentless process exits.
      wpid = wait((int *) 0);
      if(wpid == pid){
        // the shell exited; restart it.
        break;
      } else if(wpid < 0){
        printf("init: wait returned an error\n");
        exit(1);
      } else {
        // it was a parentless process; do nothing.
      }
    }
  }
}
```

`main`首先尝试以读写模式打开`console`设备，如果失败，调用`mknod`来创建设备节点；然后用`open`打开确保获取到文件描述符，这里是第一个打开的文件，所以是文件描述符0；再调用`dup`创建 stdout 和 stderr，这里实际上通过复制文件描述符0，分别得到1、2，这样就绑定了标准输入、输出和错误。

> `dup`复制给定的文件描述符并返回新的文件描述符，这两个描述符指向同一个文件表项。

Shell 首先打开文件描述符0、1、2，然后向文件描述符2打印提示符`$`。

```c
int
getcmd(char *buf, int nbuf)
{
  write(2, "$ ", 2);
  memset(buf, 0, nbuf);
  gets(buf, nbuf);
  if(buf[0] == 0) // EOF
    return -1;
  return 0;
}
```

这里代码是直接调用了`write`系统调用。也可通过`fprintf(2, "$ ")`，由 Shell 输出的每个字符都会触发一个`write`系统调用。

```c
static void
putc(int fd, char c)
{
  write(fd, &c, 1);
}
```

`write`系统调用最终会走到`sys_write`。

```c
uint64
sys_write(void)
{
  struct file *f;
  int n;
  uint64 p;

  argaddr(1, &p);
  argint(2, &n);
  if(argfd(0, 0, &f) < 0)
    return -1;

  return filewrite(f, p, n);
}
```

函数先检查参数，然后调用`filewrite`。

```c
int
filewrite(struct file *f, uint64 addr, int n)
{
  int r, ret = 0;

  if(f->writable == 0)
    return -1;

  if(f->type == FD_PIPE){
    ret = pipewrite(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].write)
      return -1;
    ret = devsw[f->major].write(1, addr, n);
  } else if(f->type == FD_INODE){
    // write a few blocks at a time to avoid exceeding
    // the maximum log transaction size, including
    // i-node, indirect block, allocation blocks,
    // and 2 blocks of slop for non-aligned writes.
    // this really belongs lower down, since writei()
    // might be writing a device like the console.
    int max = ((MAXOPBLOCKS-1-1-2) / 2) * BSIZE;
    int i = 0;
    while(i < n){
      int n1 = n - i;
      if(n1 > max)
        n1 = max;

      begin_op();
      ilock(f->ip);
      if ((r = writei(f->ip, 1, addr + i, f->off, n1)) > 0)
        f->off += r;
      iunlock(f->ip);
      end_op();

      if(r != n1){
        // error from writei
        break;
      }
      i += r;
    }
    ret = (i == n ? n : -1);
  } else {
    panic("filewrite");
  }

  return ret;
}
```

`filewrite`中首先会判断文件描述符的类型。`mknod`生成的文件描述符属于设备`FD_DEVICE`，对这种类型，会为特定设备执行设备相应的`write`函数。这里会调用`console.c`中的`consolewrite`函数。

```c
int
consolewrite(int user_src, uint64 src, int n)
{
  int i;

  for(i = 0; i < n; i++){
    char c;
    if(either_copyin(&c, user_src, src+i, 1) == -1)
      break;
    uartputc(c);
  }

  return i;
}
```

先通过`either_copyin`将字符拷入，然后调用`uartputc`函数将字符写入给 UART 设备。可以认为这个函数是一个 UART 驱动的`top`部分。

```c
void
uartputc(int c)
{
  acquire(&uart_tx_lock);

  if(panicked){
    for(;;)
      ;
  }
  while(uart_tx_w == uart_tx_r + UART_TX_BUF_SIZE){
    // buffer is full.
    // wait for uartstart() to open up space in the buffer.
    sleep(&uart_tx_r, &uart_tx_lock);
  }
  uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE] = c;
  uart_tx_w += 1;
  uartstart();
  release(&uart_tx_lock);
}
```

`uartputc`会实际打印字符。UART 内部有一个32字符大小的 buffer 用来发送数据，并有一个为 consumer 提供的读指针和为 producer 提供的写指针，构建出一个环形的 buffer。

```c
#define UART_TX_BUF_SIZE 32
char uart_tx_buf[UART_TX_BUF_SIZE];
uint64 uart_tx_w; // write next to uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE]
uint64 uart_tx_r; // read next from uart_tx_buf[uart_tx_r % UART_TX_BUF_SIZE]
```

在例子里，Shell 是 producer，`uartputc`首先判断环形 buffer 是否已满，如果已满，则会`sleep`一段时间，把 CPU 出让给其他进程。这里提示符`$`是我们送出的第一个字符，因此代码会往下走，把字符送到 buffer 中，更新写指针，然后调用`uartstart`函数。

```c
void
uartstart()
{
  while(1){
    if(uart_tx_w == uart_tx_r){
      // transmit buffer is empty.
      ReadReg(ISR);
      return;
    }

    if((ReadReg(LSR) & LSR_TX_IDLE) == 0){
      // the UART transmit holding register is full,
      // so we cannot give it another byte.
      // it will interrupt when it's ready for a new byte.
      return;
    }

    int c = uart_tx_buf[uart_tx_r % UART_TX_BUF_SIZE];
    uart_tx_r += 1;

    // maybe uartputc() is waiting for space in the buffer.
    wakeup(&uart_tx_r);

    WriteReg(THR, c);
  }
}
```

`uartstart`通知设备执行操作。首先检查当前设备是否空闲，如果空闲，就从 buffer 中读出数据，将数据写入到 THR 发送寄存器。这相当于告诉设备，有一个字节需要发送。一旦数据送到设备，系统调用就会返回，用户应用程序 Shell 就可以继续执行。这里由内核返回用户空间的机制与`trap`相同。

---

## UART驱动的bottom部分

向 Console 输出字符过程中，如果发生了中断，由于之前已经在 SSTATUS 寄存器中打开了中断，这里会触发中断。假设键盘生成了一个中断并发向 PLIC，PLIC 会将中断路由给一个特定的 CPU 核，如果该核设置了 SIE寄存器的E bit（针对外部中断），那么处理过程如下：

- 首先清除 SIE 寄存器的相应 bit 位。这样可以组织 CPU 核被其他中断打扰。

- 然后设备 SEPC 寄存器为当前的程序计数器。

- 保存当前的 mode。

- 将 mode 设置为 Supervisor mode。

- 将程序计数器的值设置为 STVEC 的值。

对于 Shell 的例子，STVEC的值为`uservec`的地址，`uservec`又会调用`usertrap`处理中断。

```c
  else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause 0x%lx pid=%d\n", r_scause(), p->pid);
    printf("            sepc=0x%lx stval=0x%lx\n", r_sepc(), r_stval());
    setkilled(p);
  }
```

在`devintr`中，首先通过 SCAUSE 寄存器判断中断是否来自外设，若是，则调用`plic_claim`获取中断。

```c
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

`plic_claim`中，当前 CPU 核会告诉 PLIC 自己要处理中断，`PLIC_SCLAIM`会将中断号返回。对于 UART来说，返回的中断号为10。

```c
int
plic_claim(void)
{
  int hart = cpuid();
  int irq = *(uint32*)PLIC_SCLAIM(hart);
  return irq;
}
```

在`devintr`函数中，如果是 UART中断，会调用`uartintr`函数。它会从 UART 的接收寄存器中读取数据，然后将获取的数据传递给`consoleintr`函数。不过这里目前还没有通过键盘输入任何数据，所以 UART 的接收寄存器为空。

```c
// read one input character from the UART.
// return -1 if none is waiting.
int
uartgetc(void)
{
  if(ReadReg(LSR) & 0x01){
    // input data is ready.
    return ReadReg(RHR);
  } else {
    return -1;
  }
}

// handle a uart interrupt, raised because input has
// arrived, or the uart is ready for more output, or
// both. called from devintr().
void
uartintr(void)
{
  // read and process incoming characters.
  while(1){
    int c = uartgetc();
    if(c == -1)
      break;
    consoleintr(c);
  }

  // send buffered characters.
  acquire(&uart_tx_lock);
  uartstart();
  release(&uart_tx_lock);
}
```

所以代码会直接运行到`uartstart`函数，这个函数会将 Shell 存储在 buffer 中的任意字符送出。这样驱动的`top`部分就和`bottom`部分就解耦开了。

> 只有一个 UART 设备，一个 buffer 只针对一个 UART 设备，而这个 buffer 会被所有 的 CPU 核共享。因此需要锁来确保多个 CPU 核上的程序串行的向 Console 打印输出。

---

## Interrupt相关的并发

与中断相关的并发包括：

- 设备与 CPU 是并行运行的。例如，当 UART 向 Console 发送字符的时候，CPU 会返回执行 Shell，而 Shell 可能会再执行一次系统调用，向 buffer 中写入另一个字符。这里的并行称为**producer-consumer**并行。

- 中断会停止当前运行的程序。对于一些内核代码来说，如果不能在执行期间被中断，则需要内核临时关闭中断，来确保这段代码的原子性。

- 驱动的 top 和 bottom 部分是并行运行的。例如，Shell 会在传输完提示符`$`之后会再调用`write`系统调用传输空格字符，代码会走到 UART 驱动的 top 部分，将空格写入到 buffer 中。但同时在另一个 CPU 核，可能会收到来自 UART 的中断，进而执行驱动的 bottom 部分，查看相同的 buffer。所以一个驱动的这两部分可以并行的在不同 CPU 上运行，通过`lock`来管理并行。

这里主要关注第一点。驱动中会有 buffer，在前面 UART的例子中，buffer 是32自己大小，且有两个指针，分别是读指针和写指针。如果两个指针相等，buffer 是空的。当 Shell 调用`uatrputc`函数时，会将字符写入写指针位置并将写指针加1。

`producer`可以一直写入数据，直到写指针+1等于读指针。这时 buffer 满了，代码会 sleep，暂时搁置 Shell 并运行其他进程。

`Interrupt handler`即`uartintr`函数在这个场景下是`consumer`。每当

有一个中断，且读指针落后于写指针，`uartintr`就会从读指针读取一个字符再通过 UART 设备发送，并将读指针加1，直到二者相等。

---

## UART读取键盘输入

Shell 会调用`read`从键盘中读取字符，在`read`系统调用的最底层，会调用`fileread`函数，这个函数中，如果读取的文件类型是设备，会调用相应设备的`read`函数。

```c
int
fileread(struct file *f, uint64 addr, int n)
{
  int r = 0;

  if(f->readable == 0)
    return -1;

  if(f->type == FD_PIPE){
    r = piperead(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].read)
      return -1;
    r = devsw[f->major].read(1, addr, n);
  } else if(f->type == FD_INODE){
    ilock(f->ip);
    if((r = readi(f->ip, 1, addr, f->off, n)) > 0)
      f->off += r;
    iunlock(f->ip);
  } else {
    panic("fileread");
  }

  return r;
}
```

在这个例子中的就是`consoleread`函数。

```c
int
consoleread(int user_dst, uint64 dst, int n)
{
  uint target;
  int c;
  char cbuf;

  target = n;
  acquire(&cons.lock);
  while(n > 0){
    // wait until interrupt handler has put some
    // input into cons.buffer.
    while(cons.r == cons.w){
      if(killed(myproc())){
        release(&cons.lock);
        return -1;
      }
      sleep(&cons.r, &cons.lock);
    }

    c = cons.buf[cons.r++ % INPUT_BUF_SIZE];

    if(c == C('D')){  // end-of-file
      if(n < target){
        // Save ^D for next time, to make sure
        // caller gets a 0-byte result.
        cons.r--;
      }
      break;
    }

    // copy the input byte to the user-space buffer.
    cbuf = c;
    if(either_copyout(user_dst, dst, &cbuf, 1) == -1)
      break;

    dst++;
    --n;

    if(c == '\n'){
      // a whole line has arrived, return to
      // the user-level read().
      break;
    }
  }
  release(&cons.lock);

  return target - n;
}
```

这里也有一个 buffer，包含128个字符。

```c
#define INPUT_BUF_SIZE 128
  char buf[INPUT_BUF_SIZE];
  uint r;  // Read index
  uint w;  // Write index
  uint e;  // Edit index
} cons;
```

在这个场景下，Shell是`consumer`，从 buffer 中读取数据；而键盘是`producer`，将数据写入到 buffer 中。

`consoleread`中可以看到，当读写指针一样时，进程会`sleep`。因此，当 Shell 打印 `$`后，如果键盘没有输入，Shell 进程会`sleep`，直到键盘有字符输入，这会被发送到主板上的 UART 芯片，产生中断后再被 PLIC 路由到某个 CPU 核，然后触发`devintr`函数，通过`uartgetc`获取相应字符，再传递给`consoleintr`函数。

默认情况下，字符会通过`consputc`输出到 console 上给用户看，然后被存放在 buffer 中。遇到换行符时，唤醒之前`sleep`的进程，再从 buffer 中将数据读出。

这样可以通过 buffer 将`consumer`和`producer`解耦，使它们可以按照自己的速度独立并行运行。

---

## Interrupt的演进

polling和Interrupt
