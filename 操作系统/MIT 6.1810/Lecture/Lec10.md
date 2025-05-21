# Multiprocessors and locking

---

## 为什么要使用锁

>[!info] 我们通过并行来提高性能，但如果并行的系统调用使用了共享的数据，我们就需要锁，而锁又会导致这些系统调用串行执行，又限制了性能。

CPU的时钟频率从2000来开始发展缓慢，导致单线程性能基本达到了极限。因此提升性能只能依赖于多核。但当一份共享数据同时被读写时，可能会出现*race condition*，因此我们需要锁。

---

## 锁如何避免*race condition*

假设现在有多个CPU核在运行，并且它们连接到同一个内存上，在同一时间调用`kfree`。`kfree`接收一个物理地址`pa`作为参数，将它作为链表的新头节点。

CPU0首先会将它的`r->next`指向当前`freelist`，但CPU1可能在CPU0只需第二条指令`kmem.kfreelist=r`前同样完成第一步，然后晚于CPU0执行第二条指令。这样CPU0想释放的`page`就丢失了。
![[Drawing 2025-05-19 14.46.56.excalidraw]]
这种问题的解决方法就是使用锁。锁是一个内核中的对象，由结构体`lock`中的字段维护：
- **acquire**：接收指向`lock`的指针为参数，它确保任何时间只会有一个进程能获取锁；
- **release**：同样接收指向`lock`的指针为参数，同一时间尝试获取锁的其他进程需等待直到持有锁的进程调用`release`函数。

二者之间的代码通常被称为`critical section`。

---

## 什么时候使用锁

>[!attention] 当两个进程访问了一个共享的数据结构，且其中一个进程会更新共享的数据结构，则需要对这个共享的数据结构加锁。

---

## 锁的特性和死锁

锁通常有三种作用：
- 锁可以避免丢失更新；
- 锁可以打包多个操作，使它们具有原子性；
- 锁可以维护共享数据结构的不变性。

死锁最简单的场景是：首先*acquire*一个锁，然后进入到*critical section*；在*critical section*中，再*acquire*同一个锁；第二个*acquire*必须要等到第一个*acquire*状态被*release*了才能继续执行，但是不继续执行的话又走不到第一个*release*，所以程序就一直卡在这了。

还有多个锁的场景：CPU1将文件从d1移到d2，CPU2正好相反将文件从d2移到d1。假设按照参数的顺序来*acquire*锁，那么CPU1会先获取d1的锁，如果程序是真正的并行运行，CPU2同时也会获取d2的锁。之后CPU1需要获取d2的锁，这里不能成功，因为CPU2现在持有锁，所以CPU1会停在这个位置等待d2的锁释放。而另一个CPU2，接下来会获取d1的锁，它也不能成功，因为CPU1现在持有锁。这也是死锁的一个例子，有时候这种场景也被称为**deadly embrace**。

这里的解决方案是：多个锁需要进行排序，所有操作必须以相同顺序获取锁。

锁同样也会破坏程序的模块化。

---

## 锁与性能

如果想获得更高的性能，需要拆分数据结构和锁。

通常来说，开发的流程是：
- 先以*coarse-grained lock*（粗粒度锁）开始；
- 再对程序进行测试，来看一下程序能否使用多核；
- 如果不可以，意味着锁存在竞争，多个进程会尝试获取同一个锁，它们会序列化的执行。

---

## XV6中UART模块对于锁的使用

从代码来看`UART`只有一个锁：

```c
// the transmit output buffer.
struct spinlock uart_tx_lock;
#define UART_TX_BUF_SIZE 32
char uart_tx_buf[UART_TX_BUF_SIZE];
uint64 uart_tx_w; // write next to uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE]
uint64 uart_tx_r; // read next from uart_tx_buf[uart_tx_r % UART_TX_BUF_SIZE]
```

可以认为`UART`模块现在是一个*coarse-grained lock*的设计，这个锁保护了`UART`的传输缓存、写指针和读指针。传输数据时，写指针会指向传输缓存的下一个空闲槽位，而读指针指向下一个需要被传输的槽位，这种模式被称为**消费者-生产者模式**。

在`userputc`函数中，首先获取锁，然后检查当前缓存是否还有空槽位，若有则将数据置于空槽位中，写指针加1，然后调用`uartstart`并释放锁。

```c
// add a character to the output buffer and tell the
// UART to start sending if it isn't already.
// blocks if the output buffer is full.
// because it may block, it can't be called
// from interrupts; it's only suitable for use
// by write().
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

在`uartstart`中，如果`uart_tx_w`不等于`uart_tx_r`，那么缓存不为空，需要处理缓存中的一些字符。锁可以确保在下一个字符写入缓存前，缓存中的字符可以被处理完。程序最后，锁也确保了一个时间只有一个CPU进程可以写入UART的寄存器THR。

```c
// if the UART is idle, and a character is waiting
// in the transmit buffer, send it.
// caller must hold uart_tx_lock.
// called from both the top- and bottom-half.
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

当UART硬件完成传输，会产生一个中断。需考虑到UART中断可能与调用`printf`的进程并行执行，我们要确保THR寄存器只有一个写入者，且传输缓存的特性不变（`uart_tx_r`指针），在中断处理函数中也要获取锁。

```c
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

---

## 自旋锁Spin lock的实现

接下来看锁如何保持**任何时间点锁的持有者都不能超过一个**的特性。这里[[操作系统/MIT 6.1810/Book/Chapter6#Code：Locks|Chapter6 Code: Locks]]写的很详细，其中提到实现的问题，需要依赖一个特殊的硬件指令来保证一次`test_and_set`操作的原子性。在RISC-V上，这个指令是`amoswap`（atomic memory swap）。

>[!note] amoswap
>该指令接收三个参数：address，寄存器r1，寄存器r2。
>指令先对address加锁，将address中的数据保存到一个临时变量tmp中，然后将r1的数据写入到地址中，再将保存在临时变量中的数据写入到r2中，最后再将地址解锁。

看代码：

```c
// Acquire the lock.
// Loops (spins) until the lock is acquired.
void
acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk))
    panic("acquire");
  
  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();
  
  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}
```

C标准库有函数`__sync_lock_test_and_set`，大部分处理器也都有`test_and_set`硬件指令。查看`kernel.asm`：

```asm
while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    80005808:	87ba                	mv	a5,a4
    8000580a:	0cf4a7af              	amoswap.w.aq	a5,a5,(s1)
    8000580e:	2781                	sext.w	a5,a5
    80005810:	ffe5                	bnez	a5,80005808 <acquire+0x1a>
```

如果锁没有被持有，它的`locked`字段是0，那么调用test_and_set将1写入locked字段，并返回`locked`字段之前的数值0，循环结束。而如果`locked`字段之前是1，那么先将之前的1读出，然后写入一个新的1，`__sync_lock_test_and_set`会返回1，表明锁之前已经被人持有了，这样的话判断语句不成立，程序会持续循环（`spin`），直到锁的`locked`字段被设置回0。

```asm
__sync_lock_release(&lk->locked);
    800058a0:	0f50000f          	fence	iorw,ow
    800058a4:	0804a02f          	amoswap.w	zero,zero,(s1)
```

对应的C代码如下：

```c
// Release the lock.
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();
  
  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  __sync_lock_release(&lk->locked);
  
  pop_off();
}
```

上面的实现中还有三个细节需要注意：
- 没有直接用`store`指令将锁的`locked`字段写为0。这是因为`store`对于不同的实现不一定是一个原子指令。
- 在`acquire`函数的最开始，会先关闭中断。`spinlock`需要处理两类并发，一是不同CPU间的并发，另一类是相同CPU上中断和普通程序间的并发。
- *memory ordering*，指令重新排序对并发场景来说是错误的。我们需要使用`memory fence`（或者叫`synchronize`指令）来确定指令的移动范围。任何在`synchronize`之前的`load`/`store`指令都不能移动到它之后。










































