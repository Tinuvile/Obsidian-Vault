# Interrupts and device drivers

驱动程序`driver`是操作系统中管理特定设备的代码。它配置设备硬件，指示设备执行操作，处理产生的中断，并与可能正在等待设备I/O的进程交互。

需要操作系统关注的设备通常被配置为可以生成中断，这是`trap`的一种。`kernel trap`处理代码会识别设备何时引发了中断，并调用驱动程序的中断处理程序。`xv6`中，这种调度发生在`devintr()`函数。

在设备驱动程序中，代码通常分为两个部分执行：一个是在进程的内核线程中运行的**Top Half**，另一个是在中断时执行的**Bottom Half**。顶部部分通过系统调用来调用，这些系统调用希望设备执行I/O操作，然后等待操作完成并引发中断。底部部分为中断处理程序，确定已经完成的操作，如果合适则唤醒等待的进程，并告诉硬件开始处理等待着的下一个操作。

---

## Code：Console input

## 代码：控制台输入

控制台驱动程序`console.c`是驱动程序结构的一个示例。它通过连接到 RISC-V 的 UART 串口硬件接受输入的字符。控制台驱动程序一次累积一行输入，用`consoleintr`处理特殊输入字符。用户进程使用`read`系统调用从控制台获取输入行，驱动程序通过 **QEMU** 模拟的 16550芯片与 UART 硬件通信。

### UART

UART 硬件对软件来说表现为一组内存映射的控制寄存器。RISC-V 将一些物理地址映射到 UART 设备，因此 load 和 store 会直接与设备硬件交互，而非 RAM 。

```c
// qemu puts UART registers here in physical memory.
#define UART0 0x10000000L
#define UART0_IRQ 10
```

UART 的内存映射地址从`0x10000000`开始。其中有一些控制寄存器，内存映射地址计算为`#define Reg(reg) ((volatile unsigned char *)(UART0 + (reg)))`。

```c
// 寄存器定义 uart.c
#define RHR 0    // 接收保持寄存器（读取输入字节）
#define THR 0    // 发送保持寄存器（写入输出字节）
#define IER 1    // 中断使能寄存器
#define IER_RX_ENABLE (1<<0)  // 接收中断使能位
#define IER_TX_ENABLE (1<<1)  // 发送中断使能位
#define FCR 2    // FIFO控制寄存器
#define FCR_FIFO_ENABLE (1<<0)  // 启用FIFO缓冲
#define FCR_FIFO_CLEAR (3<<1)  // 清除FIFO内容
#define LCR 3    // 线路控制寄存器
#define LCR_EIGHT_BITS (3<<0)  // 8位数据位配置
#define LCR_BAUD_LATCH (1<<7)  // 波特率设置模式位
#define LSR 5    // 线路状态寄存器
#define LSR_TX_IDLE (1<<5)  // 发送空闲标志（THR可接受新字符）
```

举例来说，LSR 寄存器包含指示是否有输入字符等待读取的位，这些字符从 RHR 寄存器读取，每读取一个字符，硬件会将其从等待字符的内部 FIFO 中删除，并在 FIFO 为空时清除 LSR 中的这个位。

```c
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
```

`xv6`的`main`调用`kernel/console.c/consoleinit()`来初始化 UART 硬件。UART 在接收到每个输入字节时生成接收中断，在完成发送每个输出字节时生成发送中断。

```c
void
uartinit(void)
{
  // 1. 禁用中断
  WriteReg(IER, 0x00);

  // 2. 设置波特率
  WriteReg(LCR, LCR_BAUD_LATCH);     // 进入波特率设置模式
  WriteReg(0, 0x03);                // LSB 38.4K baud
  WriteReg(1, 0x00);                // MSB

  // 3. 设置数据格式
  WriteReg(LCR, LCR_EIGHT_BITS);     // 8位数据位，无校验

  // 4. 初始化FIFO
  WriteReg(FCR, FCR_FIFO_ENABLE | FCR_FIFO_CLEAR); // 启用并清空FIFO缓冲

  // 5. 启用中断
  WriteReg(IER, IER_TX_ENABLE | IER_RX_ENABLE); // 允许收发中断

  // 6. 初始化发送缓冲区锁
  initlock(&uart_tx_lock, "uart");
}
```

### 控制台输入

`xv6 shell`通过文件描述符从控制台读取数据，`read`系统调用通过内核会到达`consoleread`。它等待输入通过中断到达并从缓冲区`cons.buf`读取字符，将输入复制到用户空间，并在整行到达后返回用户进程。

```c
int
consoleread(int user_dst, uint64 dst, int n)
{
  uint target;
  int c;
  char cbuf;

  target = n;                 // 保存原始请求读取长度
  acquire(&cons.lock);        // 获取控制台锁

  // 主循环：处理每个字符读取
  while(n > 0){
    // 等待输入缓冲区有数据（中断处理填充）
    while(cons.r == cons.w){
      if(killed(myproc())){   // 检查进程是否被终止
        release(&cons.lock);
        return -1;           // 返回错误
      }
      sleep(&cons.r, &cons.lock); // 睡眠等待数据到达
    }

    // 从循环缓冲区读取字符
    c = cons.buf[cons.r++ % INPUT_BUF_SIZE]; 

    // 处理Ctrl+D（EOF）
    if(c == C('D')){ 
      if(n < target){         // 保留EOF标记供下次读取
        cons.r--;             // 回退读指针
      }
      break;                  // 结束读取
    }

    // 复制字符到用户空间
    cbuf = c;
    if(either_copyout(user_dst, dst, &cbuf, 1) == -1)
      break;                  // 用户地址无效时终止

    dst++;  // 移动目标指针
    --n;     // 递减剩余读取计数

    // 遇到换行符提前返回
    if(c == '\n')
      break;
  }

  release(&cons.lock); // 释放控制台锁
  return target - n;    // 返回实际读取字节数
}
```

当用户输入一个字符时，UART 硬件请求 RISC-V 引发一个中断，激活`trap handler`程序，它调用`devintr`，查看 scause 寄存器，发现中断来自外部设备，然后请求 PLIC[1] 硬件单元得知哪个设备引发中断。如果是 UART，`decintr`调用`uartintr`。

```c
void
uartintr(void)
{
  // 输入处理循环
  while(1){
    int c = uartgetc();      // 从UART读取字符
    if(c == -1)              // 无更多输入时退出
      break;
    consoleintr(c);          // 将字符交给控制台处理
  }

  // 输出处理部分
  acquire(&uart_tx_lock);    // 获取发送缓冲区锁
  uartstart();               // 启动发送流程
  release(&uart_tx_lock);    // 释放锁
}
```

这个函数读取正在等待的输入字符并传递给`consoleintr`。

`consoleintr`的任务是将输入字符累积到`cons.buf`中，直到一整行到达。这个函数里面有一下退格键和其他字符的特别处理，当换行符到达，它会唤醒等待的`consoleread`。

`consoleread`在检验`cons.buf`有一整行数据后，会将其复制到用户空间，并通过系统调用机制返回用户空间。

---

## Code：Console output

## 代码：控制台输出

对连接到控制台的文件描述符进行`write`系统调用最终会到达`uartputc`。

设备驱动程序维护一个输出缓冲区`uart_tx_buf`，使得`write`进程不必等待 UART 完成发送。

```c
// the transmit output buffer.
struct spinlock uart_tx_lock;
#define UART_TX_BUF_SIZE 32
char uart_tx_buf[UART_TX_BUF_SIZE];
uint64 uart_tx_w; // write next to uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE]
uint64 uart_tx_r; // read next from uart_tx_buf[uart_tx_r % UART_TX_BUF_SIZE]
```

`uartputc`将每个字符加到缓冲区，然后调用`uartstart`启动设备传输。

```c
void
uartputc(int c)
{
  acquire(&uart_tx_lock);  // 获取发送缓冲区锁

  // 系统panic时进入死循环
  if(panicked){
    for(;;)
      ;
  }

  // 等待发送缓冲区空间（32字节循环缓冲）
  while(uart_tx_w == uart_tx_r + UART_TX_BUF_SIZE){
    sleep(&uart_tx_r, &uart_tx_lock); // 阻塞等待空间释放
  }

  // 写入缓冲区并更新写指针
  uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE] = c;
  uart_tx_w += 1;

  // 尝试立即发送缓冲内容
  uartstart();

  release(&uart_tx_lock);  // 释放锁
}
```

每次 UART 完成发送一个字节，它会生成一个中断，`uartintr`调用`uartstart`，检查设备是否真的已经完成发送，并将下一个缓冲区的字符交给设备。

```c
void
uartstart()
{
  while(1){
    if(uart_tx_w == uart_tx_r){        // 检查发送缓冲区是否为空
      ReadReg(ISR);                    // 读取中断状态寄存器（清除中断标志）
      return;                          // 退出函数
    }

    if((ReadReg(LSR) & LSR_TX_IDLE) == 0){ // 检查发送保持寄存器是否空闲
      return;                          // 等待下次中断触发
    }

    int c = uart_tx_buf[uart_tx_r % UART_TX_BUF_SIZE]; // 从环形缓冲区取字符
    uart_tx_r += 1;                    // 移动读指针

    wakeup(&uart_tx_r);                // 唤醒可能阻塞的uartputc()进程

    WriteReg(THR, c);                  // 将字符写入发送保持寄存器
  }
}
```

如果进程向控制台写入多个字节，通常第一个字节由`uartputc`调用`uartstart`发送，其余的缓冲区字节在传输完成，触发中断时由`uartintr`调用`uartstart`发送。

通过缓冲区和中断，实现了设备活动和进程活动的解耦。二者可以并发来提高性能，这种思想也被称为**I/O concurrency**。

---

## Concurrency in drivers

## 驱动程序中的并发性

`concoleread`和`consoleintr`中都有`acquire`语句，这些调用获取了一个锁，用于保护控制台驱动程序的数据结构免受并发访问。这里有三种并发危险：不同 CPU 上的两个进
程可能同时调用`consoleread`；硬件可能在一个 CPU 已经在执行`consoleread`时要求该 CPU 传递一个控制台（实际上是 UART）中断；以及硬件可能在`consoleread`执行时在另一个 CPU 上传递控制台中断。

并发性在驱动程序中需要注意的另一个例子是，一个进程正在等待设备输入，但输入到达的中断信号可能在不同的进程运行时到达。因此，中断处理程序不允许考虑当前进程的状态，必须在独立于当前进程的上下文运行。因此中断处理程序通常只做较少的工作，并唤醒**top half**来完成其他工作。

---

## Timer interrupts

## 定时器中断

`xv6`使用定时器中断来维护其时钟，并使其能在计算密集型进程间切换。切换在`usertrap`和`kerneltrap`中用`yield`实现。`xv6`对连接到RISC-V CPU 的时钟硬件编程，使其定期中断每个 CPU。

定时器中断必须在`machine mode`下处理。这种模式在没有分页的情况下执行，使用一组独立的控制寄存器。因此定时器中断的处理机制与其他的`trap`处理机制完全独立。

> [!note] 不是太懂，代码中是直接把所有异常和中断都委托给了 supervisor mode。 
> 
> ```c
> // delegate all interrupts and exceptions to supervisor mode.
> w_medeleg(0xffff);
> w_mideleg(0xffff);
> ```

在`start.c/timerinit()`中配置时钟中断。部分配置是对 CLINT (Core Local Interruptor) 硬件进行编程。

```c
void timerinit() {
  // 启用监督模式定时器中断
  w_mie(r_mie() | MIE_STIE); // 设置 mie.STIE 位

  // 启用 stimecmp 扩展
  w_menvcfg(r_menvcfg() | (1L << 63)); 

  w_mcounteren(r_mcounteren() | 2);

  // 设置首次中断触发时间（当前时间 + 1000000 ticks）
  w_stimecmp(r_time() + 1000000); 
}
```

CLINT 通过比较`stimecmp`和`time`寄存器来生成中断，当`time >= stimecmp`时触发。

另外部分是保存寄存器，在`kernelvec.S`中。

定时器中断可以在任何时候发送，内核无法禁用。因此，定时器中断处理程序必须保证不会干扰被中断的内核代码。解决方案是让处理程序请求 RISC-V 引发一个 `software interrupt`并立即返回，RISC-V通过正常的`trap`机制将`software`传递给内核，并允许内核禁用它们。

```c
void clockintr() {
  // 更新时间计数器
  if(cpuid() == 0){
    acquire(&tickslock);
    ticks++;
    wakeup(&ticks);  // 唤醒等待进程
    release(&tickslock);
  }

  // 设置下次中断（10ms 后）
  w_stimecmp(r_time() + 1000000);
}
```

---

## Real world

`xv6`允许在执行内核代码和用户程序时发送设备和定时器中断。定时器中断强制从处理程序中进行线程切换。

UART 驱动程序通过读取控制寄存器一次检索一字节的数据，这种模式称为**programmed I/O**，但这种模式速度太慢。需要高速传输大量数据的设备通常使用直接内存访问（DMA），DMA 设备硬件直接将传入数据写入 RAM，并从中读取传出数据。其驱动程序会写入控制寄存器告诉设备处理准备好的数据。

中断的 CPU 开销很高，因此高速设备会用一些技巧来减少中断的需求。一个技巧是为整个批量的传入或传出请求引发单个中断。另一个技巧是驱动程序完全禁用中断，并定期检查设备以查看是否需要关注。这种技术称为`polling`。有一些驱动会在 polling 和 interrupts 之间动态切换。
