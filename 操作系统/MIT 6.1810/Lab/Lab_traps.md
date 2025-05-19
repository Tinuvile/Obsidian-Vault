## 切换分支

```bash
tinuvile@LAPTOP-7PVP3HH3:~/xv6-labs-2024$ git fetch
tinuvile@LAPTOP-7PVP3HH3:~/xv6-labs-2024$ git checkout traps
M       kernel/sysproc.c
M       kernel/trap.c
M       kernel/vm.c
Already on 'traps'
Your branch is up to date with 'origin/traps'.
tinuvile@LAPTOP-7PVP3HH3:~/xv6-labs-2024$ make clean
rm -rf *.tex *.dvi *.idx *.aux *.log *.ind *.ilg *.dSYM *.zip *.pcap \
*/*.o */*.d */*.asm */*.sym \
user/initcode user/initcode.out user/usys.S user/_* \
kernel/kernel \
mkfs/mkfs fs.img .gdbinit __pycache__ xv6.out* \
ph barrier
```

---

## RISC-V assembly ([easy](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html))

运行`make fs.img`编译并生成汇编文件。

```nasm
0000000000000000 <g>:           // 函数g的起始标签，前面是当前段内的相对偏移地址
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int g(int x) {
   0:    1141                    addi    sp,sp,-16  // 分配栈空间：将栈指针sp下移16字节
   2:    e422                    sd    s0,8(sp)       // 保存旧的栈帧指针s0到[sp+8]位置
   4:    0800                    addi    s0,sp,16   // 设置新的栈帧基址为栈顶地址
  return x+3;
}
   6:    250d                    addiw    a0,a0,3    // 将a0寄存器（第一个参数）加3
   8:    6422                    ld    s0,8(sp)       // 恢复旧的栈帧指针
   a:    0141                    addi    sp,sp,16   // 释放栈空间
   c:    8082                    ret                // 函数返回
```

RISC-V用寄存器传递参数（`a0-a7`），当超过8个时会使用栈空间。栈分配主要用于保留一些寄存器（如`s0`）、满足栈对齐和储存局部变量（当寄存器不够）。

函数`f`的汇编与`g`类似。下面来看`main`的汇编：

```nasm
000000000000001c <main>:

void main(void) {
  1c:    1141                    addi    sp,sp,-16
  1e:    e406                    sd    ra,8(sp)         // 保存返回地址到[sp+8]
  20:    e022                    sd    s0,0(sp)
  22:    0800                    addi    s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:    4635                    li    a2,13            // 第3个参数保存到寄存器a2
  26:    45b1                    li    a1,12            // 第2个参数（计算结果）保存到寄存器a1
  28:    00001517                  auipc    a0,0x1       // 将立即数左移12位后与当前PC值相加
  2c:    82850513                  addi    a0,a0,-2008 # 850 <malloc+0xde>  // 修正地址的低12位
  // 上面两句用于得到字符串常量在数据段中的地址
  30:    688000ef                  jal    ra,6b8 <printf>  // 调用printf函数
  exit(0);
  34:    4501                    li    a0,0             // 参数0保存到寄存器a0
  36:    278000ef                  jal    ra,2ae <exit>    // 调用exit系统调用
```

> Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?  
> 哪些寄存器包含了函数的参数？例如，在 main 调用 printf 时，哪个寄存器保存了数值 13？

`a0-a7`寄存器负责存储函数的参数，在`main`中，`a2`负责保存第三个参数13。

> Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)  
> 在 main 的汇编代码中，调用函数 f 的位置在哪里？调用 g 的位置又在哪里？（提示：编译器可能会内联函数。）

内联优化会将函数调用直接替换为函数体代码，因此汇编代码中并没有显式的函数调用指令。函数`f`的调用位置在`li a1,12`，直接通过寄存器`a1`传递计算结果；函数`g`的调用位置在`addiw a0,a0,3`。

> At what address is the function printf located?  
> 函数 printf 位于哪个地址？

相对当前PC的地址为`0x6b8`。

> What value is in the register ra just after the jalr to printf in main?  
> 在 main 中用 jalr 到 printf 之后，寄存器 ra 中的值是什么？

指向下一条指令地址`0x34`。

> Run the following code.  
> 运行以下代码。
> 
> ```c
> unsigned int i = 0x00646c72;
> printf("H%x Wo%s", 57616, (char *) &i);
> ```
> 
> What is the output? [Here's an ASCII table](https://www.asciitable.com/) that maps bytes to characters.  
> 输出结果是什么？这里有一个将字节映射为字符的 ASCII 表。
> 
> The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?  
> 输出结果依赖于 RISC-V 是小端序的事实。如果 RISC-V 改为大端序，为了得到相同的输出，你需要将 `i` 设置为什么值？是否需要将 `57616` 改为不同的值？
> 
> [Here's a description of little- and big-endian](http://www.webopedia.com/TERM/b/big_endian.html) and [a more whimsical description](https://www.rfc-editor.org/ien/ien137.txt).  
> 这里有一段关于小端序和大端序的描述，以及一个更富想象力的解释。

大端序（big-endian）和小端序（little-endian）用于描述多字节数据类型中哪些字节是最高有效位，并说明字节序列在计算机内存中的存储顺序。在大端序中，序列中的最高有效值存储在最低的内存地址，小端序则相反。许多大型计算机，如IBM大型机，采用大端架构；而现代计算机，如PC之类则采用小端序。给出的后面那篇文章是这两派的交锋的详细介绍，还挺有意思，可以看一下。

`i`分解为`0x00 64 6c 72`，对应`d l r`，由于是小端序，所以应该是`rld`。`57616`的十六进制为`0xe110`。故整体输出为`He110 World`。

若为大端序，则`i`应改为`0x726c6400`，`57616`不用改。

> In the following code, what is going to be printed after `'y='`? (note: the answer is not a specific value.) Why does this happen?  
> 在以下代码中， `'y='` 之后会打印什么内容？（注意：答案不是一个具体数值。）为什么会发生这种情况？
> 
> ```c
> printf("x=%d y=%d", 3);
> ```

这有好几种可能性，不同编译器或者架构的表现可能不同。函数参数数量不匹配，这在C语言中属于**undefined behavior**。可能会输出随机数值，也可能导致程序崩溃。

## Backtrace ([moderate](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html))

> Implement a backtrace() function in kernel/printf.c. Insert a call to this function in sys_sleep, and then run bttest, which calls sys_sleep. Your output should be a list of return addresses with this form (but the numbers will likely be different):  
> 在 kernel/printf.c 中实现一个 backtrace() 函数。在 sys_sleep 中插入对该函数的调用，然后运行 bttest （它会调用 sys_sleep ）。你的输出应为一个返回地址列表，格式如下（但具体数字可能不同）：
> 
> ```bash
> backtrace:
> 0x0000000080002cda
> 0x0000000080002bb6
> 0x0000000080002898
> ```
> 
> After bttest exit qemu. In a terminal window: run addr2line -e kernel/kernel (or riscv64-unknown-elf-addr2line -e kernel/kernel) and cut-and-paste the addresses from your backtrace, like this:  
> 退出 qemu 后，在终端窗口中运行 addr2line -e kernel/kernel （或 riscv64-unknown-elf-addr2line -e kernel/kernel ），并剪切粘贴回溯中的地址，如下所示：
> 
> ```bash
> $ addr2line -e kernel/kernel
> 0x0000000080002de2
> 0x0000000080002f4a
> 0x0000000080002bfc
> Ctrl-D
> ```
> 
> You should see something like this:  
> 你应该会看到类似这样的输出：
> 
> ```bash
> kernel/sysproc.c:74
> kernel/syscall.c:224
> kernel/trap.c:85
> ```

回溯的内容是错误发生点之上栈中函数调用的列表。编译器会生成机器码，在栈上维护与当前调用链中每个函数对应的栈帧。每个栈帧由返回地址和指向调用者栈帧的指针组成。寄存器`s0`包含指向当前栈帧的指针（实际指向栈上保存的返回地址加8的位置）。`backtrace`需要利用栈指针遍历栈，并打印每个栈帧中保存的返回地址。

根据提示，在`riscv.h`中添加`r_fp`函数，可调用它利用[in-line assembly](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html)来读取`s0`。

```c
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

另外，根据提示，保存的帧指针位于帧指针固定偏移量（-16）处，返回地址位于（-8）处。

```c
void
backtrace(void)
{
  printf("backtrace:\n");

  uint64 fp = r_fp();
  uint64 page_addr = PGROUNDDOWN(fp);

  while (PGROUNDDOWN(fp) == page_addr) {
    uint64 ra = *(uint64*)(fp - 8);
    printf("%p\n", (void *)ra);
    fp = *(uint64*)(fp - 16);
  }
}
```

比较简单，然后在`panic`函数和`sys_sleep`函数中加上它的调用。

```bash
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ bttest
backtrace:
0x0000000080001d3c
0x0000000080001c5a
0x00000000800019dc 
$ QEMU: Terminated

tinuvile@LAPTOP-7PVP3HH3:~/xv6-labs-2024$ addr2line -e kernel/kernel
0x0000000080001d3c
/home/tinuvile/xv6-labs-2024/kernel/sysproc.c:58
0x0000000080001c5a
/home/tinuvile/xv6-labs-2024/kernel/syscall.c:141
0x00000000800019dc
/home/tinuvile/xv6-labs-2024/kernel/trap.c:76
```

## Alarm ([hard](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html))

> In this exercise you'll add a feature to xv6 that periodically alerts a process as it uses CPU time. This might be useful for compute-bound processes that want to limit how much CPU time they chew up, or for processes that want to compute but also want to take some periodic action. More generally, you'll be implementing a primitive form of user-level interrupt/fault handlers; you could use something similar to handle page faults in the application, for example. Your solution is correct if it passes alarmtest and 'usertests -q'  
> 在本练习中，您将为 xv6 添加一项功能，定期提醒进程其 CPU 使用时间。这对于希望限制 CPU 占用时间的计算密集型进程，或既需计算又想定期执行其他操作的进程非常有用。更广泛地说，您将实现一种用户级中断/故障处理程序的初级形式；例如，您可以使用类似机制处理应用程序中的页面错误。若您的解决方案能通过 alarmtest 和'usertests -q'测试，则视为正确。

先做**test0: invoke handler**的部分。

在`user.h`中填入声明并更新`user/usys.pl`、`kernel/syscall.h`、和`kernel/syscall.c`。允许`alarmtest`调用`sigalarm`和`sigreturn`系统调用。

然后在`struct proc`中添加新字段，分别记录警报间隔、自上次调用进程警报处理函数的时钟滴答数 ticks 和处理函数的指针。并在`proc.c/allocproc()`中初始化这些字段。

```c
int alarm_interval;          // Alarm interval
int alarm_counter;           // Alarm counter
uint64 alarm_handler;        // Alarm handler
```

函数实现：

```c
uint64
sys_sigalarm(void)
{
  int interval;
  argint(0, &interval);
  if(interval < 0)
    return -1;

  uint64 handler;
  argaddr(1, &handler);

  struct proc *p = myproc();
  p->alarm_counter = 0;
  p->alarm_interval = interval;
  p->alarm_handler = handler;

  return 0;
}

uint64
sys_sigreturn(void)
{
  return 0;
}
```

对于每个 ticks，硬件时钟会强制触发一个中断，由`usertrap()`处理。接下来修改`usertrap()`，使它能处理进程警报间隔到期的情况。

```c
if(which_dev == 2)
  useralarm(p);
```

`useralarm`的逻辑即为每次增加`alarm_count`。当它到期，设置`p->trapframe->epc`到处理函数地址，使进程从内核返回后可以执行这个函数。

```c
static int
useralarm(struct proc *p)
{
  if (p->alarm_interval == 0)
    return 0;

  p->alarm_counter++;
  if (p->alarm_counter == p->alarm_interval)
  {
    p->alarm_counter = 0;
    p->trapframe->epc = (uint64) p->alarm_handler;
    p->alarm_interval = 0;
    p->alarm_handler = 0;
  }

  return 0;
}
```

尝试运行一下测试：

```bash
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ alarmtest
test0 start
....................................................................alarm!
test0 passed
test1 start
......alarm!
..................................................................................................................................................................................................
test1 failed: too few calls to the handler
test2 start
...................................................alarm!
test2 passed
test3 start
test3 passed
```

根据文档提示继续往下，这些问题可能都是因为没有确保当警报处理程序完成后，控制权返回到用户程序最初被定时器中断打断的那条指令。

解决方案是：用户警报程序在完成时必须调用`sigreturn`系统调用来使用户进程在处理好警报后可以正确恢复执行。另外要让`usertrap`在`struct proc`中保存需要的寄存器以及其他信息。

在`struct proc`中添加两个字段，分别保存触发前用户程序计数器的值和一些寄存器。

```c
uint64 alarm_epc;            // Alarm epc
int alarm_handler_running;   // Alarm handler running
struct trapframe *alarm_trapframe; // Alarm trapframe
```

在`allocproc`和`freeproc`中需要添加`alarm_trapframe`的管理代码。

然后来写`sigreturn`的逻辑，用`memmove`恢复全部寄存器。

```c
uint64
sys_sigreturn(void)
{
  struct proc *p = myproc();

  if(p->alarm_handler_running == 0)
    return -1;

  memmove(p->trapframe, p->alarm_trapframe, sizeof(struct trapframe));

  p->alarm_handler_running = 0;

  return p->trapframe->a0;
}
```

`trap.c`也做了一点修改。

```c
if(which_dev == 2){
      if(p->alarm_interval > 0 && p->alarm_handler_running == 0) {
        p->alarm_counter++;
        if(p->alarm_counter >= p->alarm_interval) {
        memmove(p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));
        p->trapframe->epc = p->alarm_handler;
        p->alarm_counter = 0;
        p->alarm_handler_running = 1;
        }
      }
    }
```

再运行测试：

```bash
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ alarmtest
test0 start
........................................................alarm!
test0 passed
test1 start
......alarm!
.....alarm!
.....alarm!
.....alarm!
.....alarm!
.....alarm!
.....alarm!
......alarm!
.....alarm!
.....alarm!
test1 passed
test2 start
........................................................................alarm!
test2 passed
test3 start
test3 passed
```

通过。`usertests -q`也没啥问题。

## Optional challenge exercises

> Print the names of the functions and line numbers in backtrace() instead of numerical addresses ([hard](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html)).  
> 打印 backtrace() 中的函数名和行号，而非数字地址（困难）。
