# Introduction and Examples

---

## read, write, exit系统调用

例如：

```c
char buf[64];
int n = read(0, buf, sizeof(buf);
```

`read`接受三个参数：

- 第一个参数是文件描述符，指向一个之前打开的文件；

- 第二个参数是指向某段内存的指针，程序可以通过指针对应地址读取内存中的数据；

- 第三个参数是代码想读取的最大长度；

- `read`的返回值可能是读到的字节数，如果到达文件结尾则会返回`0`，如果出现错误会返回`-1`。

---

## open系统调用

```c
// open.c: create a file, write to it.

#include "kernel/types.h"
#include "user.user.h"
#include "kernel/fcntl.h"


int
main()
{
    int fd = open("output.txt", 0_WRONLY | 0_CREATE);
    write(fd, "ooo\n", 4);

    exit(0);
}
```

这个程序会创建一个`output.txt`文件并向其中写入一些数据，然后退出。其中执行了`open`系统调用，将文件名作为第一个参数传入，第二个参数是一些标志位，来告诉`open`系统调用在内核中的实现：我们要创建并写入一个文件，然后`open`系统调用会返回一个新分配的文件描述符。然后这个文件描述符作为第一个参数被传到`write`。

文件描述符本质上对应了内核中的一个表单数据，内核维护每个运行进程的状态，为每个运行进程保存一个表单，文件描述符就是这个表单的**key**，表单让内核知道每个文件描述符对应的实际内容是什么。

但是值得注意的是，每个进程都有自己独立的文件描述符空间，所以如果运行了两个不同的程序对应两个不同进程，但它们打开同一个文件，可能它们会有相同数字的文件描述符，但是因为内核为每个进程都维护一个独立文件描述符空间，所以即使是相同数字的文件描述符也可能对应不同文件。

---

## Shell

`Shell`是最传统与基础的**Unix**接口，是一个终端接口用来给用户与机器交互。

---

## fork系统调用

`fork`会拷贝当前进程的内存（进程的指令和数据），并创建一个新的进程。`fork`系统调用在两个进程中都会返回，原始进程中会返回一个大于0的整数，这个是新创建进程的**ID**，而新创建的进程中会返回0，我们通过返回值区分两个进程。这样两个进程同时运行，QEMU实际上在模拟多核模拟器。

当我们在`Shell`中运行程序时，`Shell`实际上会创建一个新进程来运行输入的每一条指令。

---

## exec, wait系统调用

`echo`命令接收传递给它的输入，并将输入写到输出。

```c
// exec.c: replace a process with an executable file

#include "kernel/types.h"
#include "user/user.h"


int
main()
{
    char *argv[] = { "echo", "this", "is", "echo", 0 };
    exec("echo", argv);
    printf("exec failed!\n");

    exit(0);
}
```

代码执行`exec`系统调用，这个系统调用会从指定文件中读取并加载指令，并替代当前调用进程的指令。这样就相当于丢弃了调用进程的内存，并开始执行新加载的指令。在这个代码中，操作系统从名为`exec`的文件中加载指令到当前进程，替换了当前进程的内存，然后开始执行这些新加载的指令，同时后面传入的是命令行参数，这里实际上就等价于运行`echo`命令。

需要注意的是：

- `exec`系统调用会保留当前的文件描述符表单，所以任何在`exec`系统调用之前的文件描述符在新程序中会表示相同的东西；

- 通常`exec`系统调用不会返回，因为会完全替换当前进程的内存。

在`Shell`中，它会先`fork`，在子进程中再`exec`，这是一个常见的**Unix**程序调用风格。

```c
#include "user/user.h"

int main() {
    int pid, status;

    pid = fork();

    if (pid == 0) {
        char *argv[] = { "/bin/echo", "THIS", "IS", "ECHO", 0 };
        exec("/bin/echo", argv);
        printf("exec failed!\n");
        exit(1);
    } else {
        printf("parent waiting\n");
        wait(&status)
        printf("the child exited with status %d\n", status);
    }

    exit(0);
}
```

`wait`系统调用会等待之前创建的子进程退出，这里`wait`的参数`status`是一种让退出的子进程以一个整数(32bit)的格式与等待的父进程通信的方式。`exit`退出的参数是1，操作系统会把1从退出的子进程传递到等待的父进程处，内核会向`status`对应的地址写入子进程传入的参数。在**Unix**中，如果一个程序成功退出，`exit`的参数会是0，如果出现错误就会向`exit`传递1，因此可以通过父进程读取`wait`的参数来了解子进程的情况。

另一个值得注意的是这里先`fork`再`exec`的写法，这个写法很常用，但实际上有些浪费，因为`fork`首先拷贝了整个父进程，但随后`exec`把这个拷贝丢弃了，这是一个浪费的操作，这些问题可以在后面通过一些优化来完成，涉及到虚拟内存系统的技巧，可以构建一个`fork`，对内存实行`lazy`拷贝，如果`fork`之后立刻是`exec`，那就不用实际的拷贝。

## I/O Redirect
