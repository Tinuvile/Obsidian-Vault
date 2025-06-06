# Isolation & system call entry/exit

---

这一节的笔记写的有点乱，Chapter4会好一些

---

## Trap机制

程序运行时会发生用户空间与内核空间的切换，每当：

- 程序执行系统调用；

- 程序出现类似`page fault`、运算除以零的错误；

- 一个设备触发了中断使得当前程序运行需要响应内核设备驱动

都会发生切换。我们把这里用户空间和内核空间的切换称为`trap`。

RISC-V有32个用户寄存器。其中需要关注的有：

- 堆栈寄存器**Stack Pointer**

- 程序计数器**Program Counter Register**

- 寄存器中表明当前`mode`的标志位

- SATP寄存器**Supervisor Address Translation and Protection**

- STVEC寄存器**Supervisor Trap Vector Base Address Register**

- SEPC寄存器**Supervisor Exception Program Counter**

- SSRATCH寄存器**Supervisor Scratch Register**

在`trap`最开始的时候，CPU处于`user mode`，因此我们要改一些状态才可以运行系统内核中的代码：

1. 首先，需要保存32个用户寄存器，因为后面需要恢复用户应用程序的执行；

2. 程序计数器同样需要被保存；

3. 然后将`mode`改为`supervisor mode`；

4. 修改SATP寄存器指向，原先指向的是`user page table`，需要转向`kernel page table`；

5. 需要将堆栈寄存器指向位于内核的一个地址，我们利用它来调用内核的函数代码；

6. 设置完成后进入内核代码。

我们仍然希望保持良好的隔离性，因此`trap`中涉及到的硬件和内核机制不能依赖任何来自用户空间的东西。

> 额外说明一下`mode`标志位寄存器：从`user mode`切换到`supervisor mode`后，实际上只能多做两件事。一是可以读写控制寄存器，即上面提到的那些寄存器；二是可以使用`PTE_U`标志位为0的`PTE`。但此模式仍然不能读写任意物理地址，它同样受限于当前`page table`设置的虚拟地址。

---

## Trap代码执行流程

我们以`Shell`中调用`write`系统调用为例。这通过执行`ecall`指令来执行。`ecall`指令会切换到`supervisor mode`的内核中。然后内核会执行`uservec`，这是一个由汇编语言写的函数，位于`trampoline.S`中；在这个汇编函数中，代码执行会跳转到`usertrap`，这个函数在`trap.c`中；在`usertrap`中，会执行一个叫做`syscall`的函数，这个函数在一个表单中根据传入的数字查找已经实现的对应的系统调用函数，对于本例是`sys_write`；`syswrite`会将要显示的数据输出到`console`上，完成后返回给`syscall`。

上述执行流程相当于在`ecall`处就中断了用户代码的执行，然后我们需要恢复它。

`syscall`函数会调用`usertrapret`函数，这个函数会完成方便在C代码中实现的返回用户空间操作；然后再运行`userret`函数，完成只能在汇编语言中完成的操作；最终，这个汇编函数会调用机器指令返回用户空间，并恢复用户程序的执行。

---

## ECALL指令之前的状态

接下来使用**gdb**来跟踪`write`系统调用，我们跟踪`Shell`将提示信息通过`write`系统调用走到操作系统再走到`console`的过程。

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

上面`write(2, "$", 2)`将`"$"`写入到文件描述符`2`。接下来打开`gdb`：

```bash
tinuvile@LAPTOP-7PVP3HH3:~/xv6-labs-2024$ make qemu-gdb
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::26000
```

然后：

```bash
tinuvile@LAPTOP-7PVP3HH3:~/xv6-labs-2024$ gdb-multiarch -q -ex "file kernel/kernel" -ex "set architecture riscv:rv64" -ex "target remote :26000"
warning: File "/home/tinuvile/xv6-labs-2024/.gdbinit" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
        add-auto-load-safe-path /home/tinuvile/xv6-labs-2024/.gdbinit
line to your configuration file "/home/tinuvile/.gdbinit".
To completely disable this security protection add
        set auto-load safe-path /
line to your configuration file "/home/tinuvile/.gdbinit".
For more information about this security protection see the
"Auto-loading safe path" section in the GDB manual.  E.g., run from the shell:
        info "(gdb)Auto-loading safe path"
Reading symbols from kernel/kernel...
The target architecture is set to "riscv:rv64".
Remote debugging using :26000
warning: Architecture rejected target-supplied description
0x0000000000001000 in ?? ()
```

当`Shell`调用`write`时，实际调用的是关联到`Shell`的一个库函数，位于`usys.S`中：

```asm6502
.global write
write:
 li a7, SYS_write
 ecall
 ret
```

它将`SYS_write`加载到`a7`寄存器中，`SYS_write`是对应数字16；然后执行`ecall`指令，代码执行跳转到内核；执行完后返回用户空间，执行`ret`，返回`Shell`中。

通过查看`xv6`编译产生的`sh.asm`寻找`ecall`指令的地址：

```asm6502
0000000000000c12 <write>:
.global write
write:
 li a7, SYS_write
     c12:    48c1                    li    a7,16
 ecall
     c14:    00000073              ecall
 ret
     c18:    8082                    ret
```

在`ecall`指令处设置断点，检查`pc`确认一下，然后打印全部32个用户寄存器：

```bash
(gdb) b *0xc12
Breakpoint 1 at 0xc12
(gdb) c
Continuing.
[Switching to Thread 1.3]

Thread 3 hit Breakpoint 1, 0x0000000000000c12 in ?? ()
(gdb) delete 1
(gdb) print $pc
$1 = (void (*)()) 0xc12
(gdb) info reg
ra             0x20     0x20
sp             0x4f70   0x4f70
gp             0x0      0x0
tp             0x0      0x0
t0             0x0      0
t1             0x0      0
t2             0x0      0
fp             0x4f90   0x4f90
s1             0x2020   8224
a0             0x2      2
a1             0x11a0   4512
a2             0x2      2
a3             0x0      0
a4             0x12     18
a5             0x2      2
a6             0x0      0
a7             0x15     21
s2             0x64     100
s3             0x20     32
s4             0x2023   8227
s5             0x12a0   4768
s6             0x0      0
s7             0x0      0
s8             0x0      0
s9             0x0      0
s10            0x0      0
s11            0x0      0
t3             0x0      0
t4             0x0      0
--Type <RET> for more, q to quit, c to continue without paging--q
Quit
```

其中`a0`、`a1`、`a2`是`Shell`传递给`write`系统调用的参数。`a0`是文件描述符，`a1`是`Shell`想写入字符串的指针，`a2`是想写入的字符数。

```bash
(gdb) x/2c $a1
0x11a0: 36 '$'  32 ' '
```

另外，上图寄存器中`pc`和`sp`的地址都在距离0比较仅的地址，这也可以进一步印证当前代码运行在用户空间中。因为用户空间中所有的地址都比较小。

系统调用时会有大量状态的变更，其中最重要的就是当前的`page table`。我们可以查看**STAP**寄存器：

```bash
(gdb) print/x $satp
$2 = 0x0
```

`0x0`表明分页功能未启用，CPU当前处于直接物理地址访问模式，未启用虚拟内存。

我们可以在QEMU中打印当前的`page table`，在QEMU界面输入`ctrl a + c`进入QEMU的`console`，然后输入`info mem`可以打印完整的`page table`。

> 这里打印的是`kernel page table`，但老师这里在用户空间🤔，不知道是不是调试模式的原因，或者是是后面`xv6`修改过，考虑到`csrrw`那个地方也不一样。

```bash
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
QEMU 7.2.0 monitor - type 'help' for more information
(qemu) info mem
vaddr            paddr            size             attr
---------------- ---------------- ---------------- -------
000000000c000000 000000000c000000 0000000000001000 rw---ad
000000000c001000 000000000c001000 0000000000001000 rw-----
000000000c002000 000000000c002000 0000000000001000 rw---ad
000000000c003000 000000000c003000 00000000001fd000 rw-----
000000000c200000 000000000c200000 0000000000001000 rw-----
000000000c201000 000000000c201000 0000000000001000 rw---ad
000000000c202000 000000000c202000 0000000000001000 rw-----
000000000c203000 000000000c203000 0000000000001000 rw---ad
000000000c204000 000000000c204000 0000000000001000 rw-----
000000000c205000 000000000c205000 0000000000001000 rw---ad
000000000c206000 000000000c206000 00000000001fa000 rw-----
000000000c400000 000000000c400000 0000000000200000 rw-----
000000000c600000 000000000c600000 0000000000200000 rw-----
000000000c800000 000000000c800000 0000000000200000 rw-----
000000000ca00000 000000000ca00000 0000000000200000 rw-----
000000000cc00000 000000000cc00000 0000000000200000 rw-----
000000000ce00000 000000000ce00000 0000000000200000 rw-----
000000000d000000 000000000d000000 0000000000200000 rw-----
000000000d200000 000000000d200000 0000000000200000 rw-----
000000000d400000 000000000d400000 0000000000200000 rw-----
000000000d600000 000000000d600000 0000000000200000 rw-----
000000000d800000 000000000d800000 0000000000200000 rw-----
000000000da00000 000000000da00000 0000000000200000 rw-----
000000000dc00000 000000000dc00000 0000000000200000 rw-----
000000000de00000 000000000de00000 0000000000200000 rw-----
000000000e000000 000000000e000000 0000000000200000 rw-----
000000000e200000 000000000e200000 0000000000200000 rw-----
000000000e400000 000000000e400000 0000000000200000 rw-----
000000000e600000 000000000e600000 0000000000200000 rw-----
000000000e800000 000000000e800000 0000000000200000 rw-----
000000000ea00000 000000000ea00000 0000000000200000 rw-----
000000000ec00000 000000000ec00000 0000000000200000 rw-----
000000000ee00000 000000000ee00000 0000000000200000 rw-----
000000000f000000 000000000f000000 0000000000200000 rw-----
000000000f200000 000000000f200000 0000000000200000 rw-----
000000000f400000 000000000f400000 0000000000200000 rw-----
000000000f600000 000000000f600000 0000000000200000 rw-----
000000000f800000 000000000f800000 0000000000200000 rw-----
000000000fa00000 000000000fa00000 0000000000200000 rw-----
000000000fc00000 000000000fc00000 0000000000200000 rw-----
000000000fe00000 000000000fe00000 0000000000200000 rw-----
0000000010000000 0000000010000000 0000000000002000 rw---ad
0000000080000000 0000000080000000 0000000000006000 r-x--a-
0000000080006000 0000000080006000 0000000000001000 r-x----
0000000080007000 0000000080007000 0000000000015000 rw---ad
000000008001c000 000000008001c000 0000000000004000 rw-----
0000000080020000 0000000080020000 0000000000001000 rw---ad
0000000080021000 0000000080021000 00000000001df000 rw-----
0000000080200000 0000000080200000 0000000000200000 rw-----
0000000080400000 0000000080400000 0000000000200000 rw-----
0000000080600000 0000000080600000 0000000000200000 rw-----
0000000080800000 0000000080800000 0000000000200000 rw-----
0000000080a00000 0000000080a00000 0000000000200000 rw-----
0000000080c00000 0000000080c00000 0000000000200000 rw-----
0000000080e00000 0000000080e00000 0000000000200000 rw-----
0000000081000000 0000000081000000 0000000000200000 rw-----
0000000081200000 0000000081200000 0000000000200000 rw-----
0000000081400000 0000000081400000 0000000000200000 rw-----
0000000081600000 0000000081600000 0000000000200000 rw-----
0000000081800000 0000000081800000 0000000000200000 rw-----
0000000081a00000 0000000081a00000 0000000000200000 rw-----
0000000081c00000 0000000081c00000 0000000000200000 rw-----
0000000081e00000 0000000081e00000 0000000000200000 rw-----
0000000082000000 0000000082000000 0000000000200000 rw-----
0000000082200000 0000000082200000 0000000000200000 rw-----
0000000082400000 0000000082400000 0000000000200000 rw-----
0000000082600000 0000000082600000 0000000000200000 rw-----
0000000082800000 0000000082800000 0000000000200000 rw-----
0000000082a00000 0000000082a00000 0000000000200000 rw-----
0000000082c00000 0000000082c00000 0000000000200000 rw-----
0000000082e00000 0000000082e00000 0000000000200000 rw-----
0000000083000000 0000000083000000 0000000000200000 rw-----
0000000083200000 0000000083200000 0000000000200000 rw-----
0000000083400000 0000000083400000 0000000000200000 rw-----
0000000083600000 0000000083600000 0000000000200000 rw-----
0000000083800000 0000000083800000 0000000000200000 rw-----
0000000083a00000 0000000083a00000 0000000000200000 rw-----
0000000083c00000 0000000083c00000 0000000000200000 rw-----
0000000083e00000 0000000083e00000 0000000000200000 rw-----
0000000084000000 0000000084000000 0000000000200000 rw-----
0000000084200000 0000000084200000 0000000000200000 rw-----
0000000084400000 0000000084400000 0000000000200000 rw-----
0000000084600000 0000000084600000 0000000000200000 rw-----
0000000084800000 0000000084800000 0000000000200000 rw-----
0000000084a00000 0000000084a00000 0000000000200000 rw-----
0000000084c00000 0000000084c00000 0000000000200000 rw-----
0000000084e00000 0000000084e00000 0000000000200000 rw-----
0000000085000000 0000000085000000 0000000000200000 rw-----
0000000085200000 0000000085200000 0000000000200000 rw-----
0000000085400000 0000000085400000 0000000000200000 rw-----
0000000085600000 0000000085600000 0000000000200000 rw-----
0000000085800000 0000000085800000 0000000000200000 rw-----
0000000085a00000 0000000085a00000 0000000000200000 rw-----
0000000085c00000 0000000085c00000 0000000000200000 rw-----
0000000085e00000 0000000085e00000 0000000000200000 rw-----
0000000086000000 0000000086000000 0000000000200000 rw-----
0000000086200000 0000000086200000 0000000000200000 rw-----
0000000086400000 0000000086400000 0000000000200000 rw-----
0000000086600000 0000000086600000 0000000000200000 rw-----
0000000086800000 0000000086800000 0000000000200000 rw-----
0000000086a00000 0000000086a00000 0000000000200000 rw-----
0000000086c00000 0000000086c00000 0000000000200000 rw-----
0000000086e00000 0000000086e00000 0000000000200000 rw-----
0000000087000000 0000000087000000 0000000000200000 rw-----
0000000087200000 0000000087200000 0000000000200000 rw-----
0000000087400000 0000000087400000 0000000000200000 rw-----
0000000087600000 0000000087600000 0000000000200000 rw-----
0000000087800000 0000000087800000 0000000000200000 rw-----
0000000087a00000 0000000087a00000 0000000000200000 rw-----
0000000087c00000 0000000087c00000 0000000000200000 rw-----
0000000087e00000 0000000087e00000 0000000000138000 rw-----
0000000087f38000 0000000087f38000 0000000000001000 rw---ad
0000000087f39000 0000000087f39000 0000000000001000 rw---a-
0000000087f3a000 0000000087f3a000 000000000000d000 rw---ad
0000000087f47000 0000000087f47000 0000000000001000 rw---a-
0000000087f48000 0000000087f48000 0000000000012000 rw---ad
0000000087f5a000 0000000087f5a000 00000000000a6000 rw-----
0000003ffff7f000 0000000087f5a000 0000000000001000 rw-----
0000003ffff81000 0000000087f5b000 0000000000001000 rw-----
0000003ffff83000 0000000087f5c000 0000000000001000 rw-----
0000003ffff85000 0000000087f5d000 0000000000001000 rw-----
0000003ffff87000 0000000087f5e000 0000000000001000 rw-----
0000003ffff89000 0000000087f5f000 0000000000001000 rw-----
0000003ffff8b000 0000000087f60000 0000000000001000 rw-----
0000003ffff8d000 0000000087f61000 0000000000001000 rw-----
0000003ffff8f000 0000000087f62000 0000000000001000 rw-----
0000003ffff91000 0000000087f63000 0000000000001000 rw-----
0000003ffff93000 0000000087f64000 0000000000001000 rw-----
0000003ffff95000 0000000087f65000 0000000000001000 rw-----
0000003ffff97000 0000000087f66000 0000000000001000 rw-----
0000003ffff99000 0000000087f67000 0000000000001000 rw-----
0000003ffff9b000 0000000087f68000 0000000000001000 rw-----
0000003ffff9d000 0000000087f69000 0000000000001000 rw-----
0000003ffff9f000 0000000087f6a000 0000000000001000 rw-----
0000003ffffa1000 0000000087f6b000 0000000000001000 rw-----
0000003ffffa3000 0000000087f6c000 0000000000001000 rw-----
0000003ffffa5000 0000000087f6d000 0000000000001000 rw-----
0000003ffffa7000 0000000087f6e000 0000000000001000 rw-----
0000003ffffa9000 0000000087f6f000 0000000000001000 rw-----
0000003ffffab000 0000000087f70000 0000000000001000 rw-----
0000003ffffad000 0000000087f71000 0000000000001000 rw-----
0000003ffffaf000 0000000087f72000 0000000000001000 rw-----
0000003ffffb1000 0000000087f73000 0000000000001000 rw-----
0000003ffffb3000 0000000087f74000 0000000000001000 rw-----
0000003ffffb5000 0000000087f75000 0000000000001000 rw-----
0000003ffffb7000 0000000087f76000 0000000000001000 rw-----
0000003ffffb9000 0000000087f77000 0000000000001000 rw-----
0000003ffffbb000 0000000087f78000 0000000000001000 rw-----
0000003ffffbd000 0000000087f79000 0000000000001000 rw-----
0000003ffffbf000 0000000087f7a000 0000000000001000 rw-----
0000003ffffc1000 0000000087f7b000 0000000000001000 rw-----
0000003ffffc3000 0000000087f7c000 0000000000001000 rw-----
0000003ffffc5000 0000000087f7d000 0000000000001000 rw-----
0000003ffffc7000 0000000087f7e000 0000000000001000 rw-----
0000003ffffc9000 0000000087f7f000 0000000000001000 rw-----
0000003ffffcb000 0000000087f80000 0000000000001000 rw-----
0000003ffffcd000 0000000087f81000 0000000000001000 rw-----
0000003ffffcf000 0000000087f82000 0000000000001000 rw-----
0000003ffffd1000 0000000087f83000 0000000000001000 rw-----
0000003ffffd3000 0000000087f84000 0000000000001000 rw-----
0000003ffffd5000 0000000087f85000 0000000000001000 rw-----
0000003ffffd7000 0000000087f86000 0000000000001000 rw-----
0000003ffffd9000 0000000087f87000 0000000000001000 rw-----
0000003ffffdb000 0000000087f88000 0000000000001000 rw-----
0000003ffffdd000 0000000087f89000 0000000000001000 rw-----
0000003ffffdf000 0000000087f8a000 0000000000001000 rw-----
0000003ffffe1000 0000000087f8b000 0000000000001000 rw-----
0000003ffffe3000 0000000087f8c000 0000000000001000 rw-----
0000003ffffe5000 0000000087f8d000 0000000000001000 rw-----
0000003ffffe7000 0000000087f8e000 0000000000001000 rw-----
0000003ffffe9000 0000000087f8f000 0000000000001000 rw-----
0000003ffffeb000 0000000087f90000 0000000000001000 rw-----
0000003ffffed000 0000000087f91000 0000000000001000 rw-----
0000003ffffef000 0000000087f92000 0000000000001000 rw-----
0000003fffff1000 0000000087f93000 0000000000001000 rw-----
0000003fffff3000 0000000087f94000 0000000000001000 rw-----
0000003fffff5000 0000000087f95000 0000000000001000 rw-----
0000003fffff7000 0000000087f96000 0000000000001000 rw-----
0000003fffff9000 0000000087f97000 0000000000001000 rw-----
0000003fffffb000 0000000087f98000 0000000000001000 rw---ad
0000003fffffd000 0000000087f99000 0000000000001000 rw---ad
0000003ffffff000 0000000080006000 0000000000001000 r-x--a-
```

`attr`列是PTE的标志位。`rwx`表示这个`page`可读可写也可执行指令，`u`表示PTE_U是否被设置，再然后是`Global`标志位，`a`表明PTE是否被使用过，`d`表明是否被写过。

另外，最后两条PTE的虚拟地址非常大，接近虚拟地址的顶端，分别是`trapframe`和`trampoline`。

接下来在`Shell`中打印出`write`的内容：

```bash
(gdb) x/3i 0xc12
=> 0xc12:       li      a7,16
   0xc14:       ecall
   0xc18:       ret
```

---

## ECALL指令之后的状态

现在执行`ecall`指令：

```bash
(gdb) stepi
0x0000000000000c14 in ?? ()
(gdb) print $pc
$3 = (void (*)()) 0xc14
```

看一下接下来要运行的指令：

```bash
(gdb) x/6i 0xc14
=> 0xc14:       ecall
   0xc18:       ret
   0xc1a:       li      a7,21
   0xc1c:       ecall
   0xc20:       ret
   0xc22:       li      a7,6
```

`ecall`会将代码从`user mode`改到`supervisor mode`，然后将程序计数器的值保存到SEPC寄存器，再跳转到STVEC寄存器指向的指令。

接下来我们还需要进行：

- 保存32个用户寄存器的内容；

- 从`user page table`切换到`kernel page table`；

- 需要将Stack Pointer寄存器指向一个`kernel stack`，给C代码提供栈；

- 需要跳转到内核中C代码的特定位置。

基于RISC-V的设计理念，这些交给软件来完成。

> RISC-V认为，`ecall`只完成尽量少且必须完成的工作，其他都交给软件完成。这样可以为软件和操作系统程序员提供最大的灵活性。

---

## uservec函数

现在程序位于`trampoline page`的起始处，也是`uservec`函数的开始。由于RISC-V中`supervisor mode`下的代码不允许直接访问物理内存，因此在拷贝用户寄存器时，只能使用`page table`而不能直接使用物理内存。另一种思路是在`supervisor mode`下更改SATP寄存器，使其指向`kernel page table`，但是在当前位置我们并不知道`kernel page table`的地址。

`xv6`的实现包含两个部分。

第一个部分里，`xv6`在每个`user page table`映射了`trapframe page`，这是由`kernel`设置的，映射指向一个可以用来存放这个进程的用户寄存器的内存位置，这个位置的虚拟地址是固定的。

```c
// per-process data for the trap handling code in trampoline.S.
// sits in a page by itself just under the trampoline page in the
// user page table. not specially mapped in the kernel page table.
// uservec in trampoline.S saves user registers in the trapframe,
// then initializes registers from the trapframe's
// kernel_sp, kernel_hartid, kernel_satp, and jumps to kernel_trap.
// usertrapret() and userret in trampoline.S set up
// the trapframe's kernel_*, restore user registers from the
// trapframe, switch to the user page table, and enter user space.
// the trapframe includes callee-saved user registers like s0-s11 because the
// return-to-user path via usertrapret() doesn't return through
// the entire kernel call stack.
struct trapframe {
  /*   0 */ uint64 kernel_satp;   // kernel page table
  /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
  /*  16 */ uint64 kernel_trap;   // usertrap()
  /*  24 */ uint64 epc;           // saved user program counter
  /*  32 */ uint64 kernel_hartid; // saved kernel tp
  /*  40 */ uint64 ra;
  /*  48 */ uint64 sp;
  /*  56 */ uint64 gp;
  /*  64 */ uint64 tp;
  /*  72 */ uint64 t0;
  /*  80 */ uint64 t1;
  /*  88 */ uint64 t2;
  /*  96 */ uint64 s0;
  /* 104 */ uint64 s1;
  /* 112 */ uint64 a0;
  /* 120 */ uint64 a1;
  /* 128 */ uint64 a2;
  /* 136 */ uint64 a3;
  /* 144 */ uint64 a4;
  /* 152 */ uint64 a5;
  /* 160 */ uint64 a6;
  /* 168 */ uint64 a7;
  /* 176 */ uint64 s2;
  /* 184 */ uint64 s3;
  /* 192 */ uint64 s4;
  /* 200 */ uint64 s5;
  /* 208 */ uint64 s6;
  /* 216 */ uint64 s7;
  /* 224 */ uint64 s8;
  /* 232 */ uint64 s9;
  /* 240 */ uint64 s10;
  /* 248 */ uint64 s11;
  /* 256 */ uint64 t3;
  /* 264 */ uint64 t4;
  /* 272 */ uint64 t5;
  /* 280 */ uint64 t6;
};
```

以上就是`xv6`在`tramframe page`中存放的内容。最开始的五个数据是内核预先存放的数据，如第一个保存了`kernel page table`的地址，这是`trap`处理代码将要加载到SATP寄存器中的数。

另一部分是SSCRATCH寄存器。在进入到`user page`之前，内核会将`trapframe page`的地址保存在这个寄存器中。

---

## usertrap函数

`usertrap`首先更改STVEC寄存器，将其指向`kernelvec`变量，这是内核空间`trap`处理代码的位置；然后保存用户程序计数器到SEPC寄存器中；接下来根据触发`trap`的原因，设置SCAUSE寄存器的值；然后就是一些处理操作，在Chapter4中有介绍。

---

## usertrapret函数

`usertrap`最后会调用`usertrapret`函数来设置返回用户空间前内核要做的工作。它首先会关闭中断，因为我们要更新STVEC寄存器指向用户空间的`trap`处理代码；然后设置STVEC寄存器指向`trampoline`代码，这里最终会执行`sret`指令返回用户空间，该指令又会重新打开中断；接下来会填充一些`trapframe`的内容，包括`kernel page table`的指针、储存了当前用户进程的`kernel stack`、储存了`usertrap`函数的指针、从`tp`寄存器中读取的CPU核编号。

`sret`指令会将程序计数器设置成SEPC寄存器的值，并根据`user page table`地址生成相应的SATP值，这样在返回用户空间时才能完成`page table`的切换。

---

## userret函数

第一步是切换`page table`，切换到`user page table`；然后将SSCRATCH寄存器恢复成保存好的用户的`a0`寄存器，这里`a0`是`trapframe`的地址。

`sret`是在`kernel`中的最后一条指令，执行完后就会回到`user mode`。
