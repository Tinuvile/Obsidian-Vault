# Calling conventions and stack frames RISC-V

---

注：这里2024版的课程顺序做了调整，可以稍微注意一下。

---

## C程序到汇编程序的转换

处理器可以理解的是二进制编码后的汇编代码。而一个RISC-V处理器可以理解RISC-V的指令集。在运行任何编译型语言前都要先生成汇编语言。

> 任何一个处理器都有一个关联的ISA（Instruction Sets Architecture），ISA就是处理器能理解的指令集。每一条指令都有一个对应的二进制编码或者一个Opcode，处理器通过这些编码就知道要进行什么操作。

---

## RISC-V vs x86

不同处理器的指令集不一样，汇编语言也不一样。现代大部分计算机都运行在**x86**和**x86-64**处理器上。RISC-V中的**RISC**是精简指令集（Reduced Instruction Set Computer）的意思，而**x86**被称为**CISC**（Complex Instruction Set Computer）复杂指令集。

---

## gdb和汇编代码执行

这里主要展示了一些gdb命令的使用：

- `tui enable`可以打开源代码展示窗口；

- `layout asm`可以看到汇编指令；

- `layout reg`可以看到寄存器信息。

---

## RISC-V寄存器

寄存器是CPU或者处理器上预先定义的可以用来存储数据的位置。寄存器之所以重要，是因为汇编代码并不是在内存里执行，而是在寄存器上执行。下面是RISC-V的寄存器表。

![2025-04-02-14-53-21-image.png](..\image\2025-04-02-14-53-21-image.png)

通常我们使用寄存器的**ABI Name**，表中第一列的寄存器名唯一重要的用处是在压缩指令（Compressed Instruction）中。通常RISC-V的指令是64bit，但压缩指令集中指令是16bit。

最后一列对于寄存器也非常重要：

- **Caller**寄存器在函数调用的时候不会保存；

- **Callee**寄存器在函数调用的时候会保存。

不会保存的意思就是可能被其他函数重新。例如**Return address**寄存器（ra），这个寄存器用来保存函数返回的地址，当函数a调用函数b的时候，b会重新ra寄存器。因此，对于一个Caller Saved寄存器，作为调用方的函数需要小心可能的数据变化；而对于一个Callee寄存器，作为被调用方的函数要小心寄存器的值不会发生相应的变化。

---

## Stack

每次我们调用一个函数，它会为自己创建一个栈帧（**Stack Frame**），并且只给自己用。函数通过移动 **Stack Pointer** 来完成 Stack Frame 的空间分配。

 Stack Pointer 总是做减法，因为栈是向下增长的，从高地址向低地址。

一个函数的 Stack Frame 包含保存的寄存器，本地变量。但当函数的参数超过8个，额外的参数就会出现在 Stack 中，因此 Stack Frame 的大小并不总是相同。但**Return address**总是出现在 Stack Frame 的第一位，另外指向前一个 Stack Frame 的指针也会出现在栈中的固定位置。

跟 Stack Frame 有关的重要寄存器有SP（Stack Pointer）和FP（Frame Pointer）。SP指向 Stack 的底部并代表当前Stack Frame的位置，FP指向当前 Stack Frame 的顶部。

通常在汇编代码中，最开始是**Function prologue**，然后是函数本体，最后是**Epilogue**：

```asm6502
0000000080001cae <sys_exit>:
#include "spinlock.h"
#include "proc.h"

uint64
sys_exit(void)
{
    80001cae:    1101                    addi    sp,sp,-32
    80001cb0:    ec06                    sd    ra,24(sp)
    80001cb2:    e822                    sd    s0,16(sp)
    80001cb4:    1000                    addi    s0,sp,32
  int n;
  argint(0, &n);
    80001cb6:    fec40593              addi    a1,s0,-20
    80001cba:    4501                    li    a0,0
    80001cbc:    ef3ff0ef              jal    ra,80001bae <argint>
  exit(n);
    80001cc0:    fec42503              lw    a0,-20(s0)
    80001cc4:    f16ff0ef              jal    ra,800013da <exit>
  return 0;  // not reached
}
    80001cc8:    4501                    li    a0,0
    80001cca:    60e2                    ld    ra,24(sp)
    80001ccc:    6442                    ld    s0,16(sp)
    80001cce:    6105                    addi    sp,sp,32
    80001cd0:    8082                    ret
```

这里`addi sp,sp,-32`首先对`Stack Pointer`减32，为`Stack Frame`创建了32字节的空间，然后`sd ra,24(sp)`将 Return address 保存在偏移24字节的位置；最后`ld ra,24(sp)`恢复返回地址，然后`addi sp,sp,32`释放栈空间，`ret`退出函数。

---

## Struct

 struct 在内存中是一段连续的地址，可以将其理解成不同字段类型可以不一样的数组。
