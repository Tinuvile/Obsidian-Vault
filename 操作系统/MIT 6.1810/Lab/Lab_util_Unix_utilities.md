## sleep (easy)

> Implement a user-level sleep program for xv6, along the lines of the UNIX sleep command. Your sleep should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file user/sleep.c.
> 
> 实现一个用户级的 sleep 程序，类似于 UNIX 的 sleep 命令。你的 sleep 应该暂停用户指定的滴答数。滴答是由 xv6 内核定义的时间概念，即来自定时器芯片的两个中断之间的时间。你的解决方案应该在文件 user/sleep.c 中。

通过阅读其他代码了解命令行参数的传递，`argc`为参数的个数，`argv`为具体的参数内容。

```c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  if (argc != 2) {  // 强制两个参数：程序名sleep + 休眠时间
    fprintf(2, "Usage: sleep ticks\n");
    exit(1);
  }

  int ticks = atoi(argv[1]);  // atoi用于将字符串转换成整数
  sleep(ticks);

  exit(0);
}
```

## pingpong (easy)

> Write a user-level program that uses xv6 system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "<pid>: received ping", where <pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "<pid>: received pong", and exit. Your solution should be in the file user/pingpong.c.  
> 编写一个用户级程序，使用 xv6 系统调用在一对管道之间“乒乓”传输一个字节，每个方向一个管道。父进程应向子进程发送一个字节；子进程应打印“pid: received ping”，其中pid是其进程 ID，将字节写入管道给父进程，然后退出；父进程应从子进程读取字节，打印“pid: received pong”，然后退出。你的解决方案应放在文件 user/pingpong.c 中。

```c
#include "kernel/types.h"
#include "user/user.h"

int
main()
{
    int parent_fd[2];
    int child_fd[2];
    char buf[1];

    if (pipe(parent_fd) < 0 || pipe(child_fd) < 0) {
        fprintf(2, "pipe error\n");
        exit(1);
    }

    int pid = fork();
    if (pid < 0) {
        fprintf(2, "fork error\n");
        exit(1);
    } else if (pid == 0) {
        // child
        close(parent_fd[1]);
        close(child_fd[0]);

        if (read(parent_fd[0], buf, sizeof(buf)) != 1) {
            fprintf(2, "read error\n");
            exit(1);
        }

        printf("%d: received ping\n", getpid());

        if (write(child_fd[1], buf, sizeof(buf))!= 1) {
            fprintf(2, "write error\n");
            exit(1);
        }

        close(parent_fd[0]);
        close(child_fd[1]);
        exit(0);
    } else {
        // parent
        close(parent_fd[0]);
        close(child_fd[1]);

        if (write(parent_fd[1], buf, sizeof(buf))!= 1) {
            fprintf(2, "write error\n");
            exit(1);
        }

        if (read(child_fd[0], buf, sizeof(buf))!= 1) {
            fprintf(2, "read error\n");
            exit(1);
        }

        printf("%d: received pong\n", getpid());

        close(parent_fd[1]);
        close(child_fd[0]);
        exit(0);
    }
}
```

## primes (moderate/hard)

> Write a concurrent prime sieve program for xv6 using pipes and the design illustrated in the picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text. This idea is due to Doug McIlroy, inventor of Unix pipes. Your solution should be in the file user/primes.c.  
> 为 xv6 编写一个并发素数筛程序，使用管道和本页中间图片及周围文字所示的设计。这个想法归功于 Unix 管道的发明者 Doug McIlroy。你的解决方案应该在文件 user/primes.c 中。

这个任务需要利用`pipe`与`fork`设置管道，第一个进程负责将数字2-280输入管道，然后对于每个质数，需要创建一个进程，该进程从左侧读取数据，并通过管道向右侧写入数据，将这个质数的倍数过滤掉。

父进程部分使用循环输入数据：

```c
// parent
close(p[0]);
for (int i = 2; i <= 280; i++) {
    if (write(p[1], &i, sizeof(i))) {
        fprintf(2, "write error\n");
        exit(1);
    }
}
close(p[1]);
wait(0);
exit(0);
```

对于这种多个输入，内核会管理管道的缓冲区，无需用户进行空间内存分配。另外，读取也是使用循环读取，管道中实际上是字节流，4字节的读即可。

根据这个过程，进入子进程中的第一个数始终是素数，可以使用数学归纳法证明。

子进程部分：

```c
// child
close(p[1]);  // 关闭写端

int prime;
while (read(p[0], &prime, sizeof(prime)) == sizeof(prime)) {
    printf("prime %d\n", prime);

    int new_p[2];  // 创建新管道
    pipe(new_p);

    int num;       // 过滤素数
    while(read(p[0], &num, sizeof(num)) == sizeof(num)) {
        if (num % prime != 0) {
            if (write(new_p[1], &num, sizeof(num))) {
                fprintf(2, "write error\n");
                exit(1);
            }
        }
    }
    close(new_p[1]);
}
close(p[0]);
exit(0);
```

需要把这部分抽象成函数，代码：

```c
#include "kernel/types.h"
#include "user/user.h"

void child(int p[2]);

int
main()
{
    int p[2];

    if (pipe(p) < 0) {
        fprintf(2, "pipe error\n");
        exit(1);   
    }

    int pid = fork();
    if (pid < 0) {
        fprintf(2, "fork error\n");
        exit(1);
    } else if (pid == 0) {
        child(p);
    } else {
        // parent
        close(p[0]);
        for (int i = 2; i <= 280; i++) {
            if (write(p[1], &i, sizeof(i)) != sizeof(i)) {
                fprintf(2, "write error\n");
                exit(1);
            }
        }
        close(p[1]);
        wait(0);
        exit(0);
    }
}

void child(int p[2]) {
    // child
    close(p[1]);

    int prime;
    if (read(p[0], &prime, sizeof(prime)) != sizeof(prime)) {
        close(p[0]);
        exit(0);
    }
    printf("prime %d\n", prime);

    int new_p[2];
    pipe(new_p);

    int pid = fork();
    if (pid < 0) {
        fprintf(2, "fork error\n");
        exit(1);
    } if (pid == 0) {
        close(new_p[1]);
        child(new_p);
    } else {
        close(new_p[0]);
        int num;

        while (read(p[0], &num, sizeof(num)) == sizeof(num)) {
            if (num % prime != 0) {
                if (write(new_p[1], &num, sizeof(num)) != sizeof(num)) {
                    fprintf(2, "write error\n");
                    exit(1);
                }
            }
        }
        close(new_p[1]);
        close(p[0]);
        wait(0); 
        exit(0);
    }
}
```

但是结果出现错误，后期乱码，怀疑是某个管道未正确关闭。

```powershell
tinuvile@LAPTOP-7PVP3HH3:~/xv6-labs-2024$ make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ primes
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
prime 37
$ /5;=CGIOSYaegkmq�����������������������
$QEMU: Terminated
```

DeepSeek认为可能原因有：

- 递归深度过大，目前的代码思路是每个质数生成一个子进程形成递归链。递归层级可能超过`xv6`默认的栈大小，导致栈溢出；

- 文件描述符泄露，未正确关闭管道文件描述符，可能耗尽`fd`资源；或可能某些分支遗漏关闭操作；

经过多次修改后成功的代码：

```c
#include "kernel/types.h"
#include "user/user.h"

void child(int p[2]);

int
main()
{
    int p[2];

    if (pipe(p) < 0) {
        fprintf(2, "pipe error\n");
        exit(1);   
    }

    int pid = fork();
    if (pid < 0) {
        fprintf(2, "fork error\n");
        exit(1);
    } else if (pid == 0) {
        child(p);
    } else {
        // parent
        close(p[0]);
        for (int i = 2; i <= 280; i++) {
            if (write(p[1], &i, sizeof(i)) != sizeof(i)) {
                fprintf(2, "write error\n");
                exit(1);
            }
        }
        close(p[1]);
        wait(0);
        exit(0);
    }
}

void child(int p[2]) {
    // child
    close(p[1]);

    int prime;
    if (read(p[0], &prime, sizeof(prime)) != sizeof(prime)) {
        close(p[0]);
        exit(0);
    }
    printf("prime %d\n", prime);

    int new_p[2];
    pipe(new_p);

    int pid = fork();
    if (pid < 0) {
        fprintf(2, "fork error\n");
        exit(1);
    } if (pid == 0) {
        close(p[0]);
        close(new_p[1]);
        child(new_p);
        exit(0);
    } else {
        close(new_p[0]);
        int num;

        while (read(p[0], &num, sizeof(num)) == sizeof(num)) {
            if (num % prime != 0) {
                if (write(new_p[1], &num, sizeof(num)) != sizeof(num)) {
                    fprintf(2, "write error\n");
                    exit(1);
                }
            }
        }
        close(new_p[1]);
        close(p[0]);
        wait(0);
    }
    exit(0);
}
```

错误原因应该在代码的`if (pid == 0)`下的`close(p[0])`这句，关闭原始管道读端并确保子进程中止。

## find (moderate)

> Write a simple version of the UNIX find program for xv6: find all the files in a directory tree with a specific name. Your solution should be in the file user/find.c.  
> 为 xv6 编写一个简单的 UNIX find 程序版本：在目录树中查找具有特定名称的所有文件。你的解决方案应该在文件 user/find.c 中。

首先理清思路。`main`函数处理入口，主要是判断参数数量是否符合要求，然后进入具体函数执行；`find`函数是提取出来的目录遍历逻辑与递归进入子目录的调用，这部分要求提示我们参照`ls.c`，那么首先阅读`ls.c`：

```c
void
ls(char *path)
{
  char buf[512], *p;  // 路径缓冲区（最大512字节）
  int fd;             // 文件描述符
  struct dirent de;   // 目录条目结构体（DIRSIZ=14字节）
  struct stat st;     // 文件状态结构体

  // 打开并验证路径
  if((fd = open(path, O_RDONLY)) < 0){ // 只读模式打开
    fprintf(2, "ls: cannot open %s\n", path);
    return;
  }

  // 获取文件状态
  if(fstat(fd, &st) < 0){  // 通过文件描述符获取状态
    fprintf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  // 根据文件类型处理
  switch(st.type){
  case T_DEVICE:      // 设备文件
  case T_FILE:        // 普通文件
    // 使用 fmtname 格式化文件名后输出元数据
    printf("%s %d %d %d\n", fmtname(path), st.type, st.ino, (int) st.size);
    break;

  case T_DIR:         // 目录文件
    // 检查路径长度是否溢出缓冲区
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("ls: path too long\n");
      break;
    }

    // 构建基础路径：将原始路径复制到buf，追加'/'
    strcpy(buf, path);
    p = buf+strlen(buf);
    *p++ = '/';       // 现在p指向路径末尾的'/'

    // 遍历目录条目
    while(read(fd, &de, sizeof(de)) == sizeof(de)){
      if(de.inum == 0) // 跳过空闲inode条目
        continue;

      // 拼接完整路径：buf = path + "/" + de.name
      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;  // 确保字符串终止

      // 获取条目状态并输出
      if(stat(buf, &st) < 0){
        printf("ls: cannot stat %s\n", buf);
        continue;
      }
      printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, (int) st.size);
    }
    break;
  }
  close(fd); // 关闭文件描述符
}
```

`find`的大部分可以参照上面实现。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
#include "kernel/fcntl.h"

void 
find(char *path, char *target) {
  char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, O_RDONLY)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    if(st.type != T_DIR){
        fprintf(2, "find: %s is not a directory\n", path);
        close(fd);
        return;
    }

    strcpy(buf, path);
    p = buf + strlen(buf);
    *p++ = '/';

    while(read(fd, &de, sizeof(de)) == sizeof(de)){
        if(de.inum == 0 || strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
            continue;

        memmove(p, de.name, DIRSIZ);
        p[DIRSIZ] = 0;

        if(stat(buf, &st) < 0){
            fprintf(2, "find: cannot stat %s\n", buf);
            continue;
        }

        // 递归处理目录或匹配文件
        if(st.type == T_DIR){
            find(buf, target);  // 递归处理子目录
        } else if(st.type == T_FILE) {
            if(strcmp(de.name, target) == 0)  // 精确匹配文件名
                printf("%s\n", buf);          // 输出完整路径
        }
    }
    close(fd);
}

int 
main(int argc, char *argv[]) {
  if(argc != 3){
    fprintf(2, "Usage: find <directory> <filename>\n");
    exit(1);
  }
  find(argv[1], argv[2]);
  exit(0);
}
```

运行测试：

```powershell
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ echo > b
$ mkdir a
$ echo > a/b 
$ mkdir a/aa
$ echo > a/aa/b
$ find . b
./b
./a/b
./a/aa/b
```

## xargs (moderate)

> Write a simple version of the UNIX xargs program for xv6: its arguments describe a command to run, it reads lines from the standard input, and it runs the command for each line, appending the line to the command's arguments. Your solution should be in the file user/xargs.c.  
> 为 xv6 编写一个简单的 UNIX xargs 程序版本：其参数描述了要运行的命令，它从标准输入读取行，并为每一行运行命令，将该行附加到命令的参数中。你的解决方案应该在文件 user/xargs.c 中。

首先解析一下命令的输入。在操作系统的视角中，命令行参数和标准输入是两套不同的机制。以这个命令为例：

```bash
echo hello | xargs echo bye
```

后面的部分属于命令行参数，通过`argv`数组解析；即`argv[0]` = "xargs"（程序名），`argv[1]` = "echo"（要执行的命令），`argv[2]` = "bye"（命令的初始参数）；而前面的部分通过标准输入读取，即文件描述符`0`，通过管道接受"hello"。

那么在这个函数中我们需要做的，一是检测命令行参数是否足够，二是将命令行与标准输入进行拼接，三是调用子进程并运行命令，使用`exec`函数替换当前进程的镜像为新的程序来运行。

详细的实现如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
#include "kernel/fcntl.h"
#include "kernel/param.h"

#define MAX_LINE 1024

int
main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(2, "xargs: not enough arguments\n");
        exit(1);
    }

    char line[MAX_LINE];
    char *cmd_argv[MAXARG];
    char ch;
    int pos = 0;

    for (int i = 1; i < argc; i++) {
        cmd_argv[i - 1] = argv[i];
    }

    while (read(0, &ch, 1) > 0) {
        if (ch == '\n') {
            if (pos == 0) {
                continue;
            }
            line[pos] = 0;
            cmd_argv[argc - 1] = line;
            cmd_argv[argc] = 0;

            if (fork() == 0) {
                exec(cmd_argv[0], cmd_argv);
                fprintf(2, "xargs: exec %s failed\n", cmd_argv[0]);
                exit(1);
            } else {
                wait(0);
                pos = 0;
            }
        } else {
            if (pos < MAX_LINE - 1) {
                line[pos++] = ch;
            }
        }
    }
    exit(0);
}
```

运行测试：

```bash
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ sh < xargstest.sh
$ $ $ $ $ $ hello
hello
hello
```

通过。

那么这个Lab除了一些拓展练习就全部写完了，拓展练习有时间再写吧。

## Optional challenge exercises

> - Write an uptime program that prints the uptime in terms of ticks using the uptime system call. ([easy](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html))  
>   编写一个 uptime 程序，使用 uptime 系统调用以滴答数打印运行时间。（简单）
> 
> - Support regular expressions in name matching for find. grep.c has some primitive support for regular expressions. ([easy](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html))  
>   在名称匹配中支持正则表达式。 grep.c 对正则表达式有一些基本的支持。（简单）
> 
> - The xv6 shell (`user/sh.c`) is just another user program. It lacks many features found in real shells, but you can modify and improve it. For example, modify the shell to not print a `$` when processing shell commands from a file ([moderate](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html)), modify the shell to support wait ([easy](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html)), modify the shell to support tab completion ([easy](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html)), modify the shell to keep a history of passed shell commands ([moderate](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html)), or anything else you would like your shell to do. (If you are very ambitious, you may have to modify the kernel to support the kernel features you need; xv6 doesn't support much.)  
>   xv6 shell（ user/sh.c ）只是另一个用户程序。它缺少真实 shell 中的许多功能，但你可以修改和改进它。例如，修改 shell 使其在处理来自文件的 shell 命令时不打印 $ （中等难度），修改 shell 以支持 wait（简单），修改 shell 以支持 tab 补全（简单），修改 shell 以保留已执行 shell 命令的历史记录（中等难度），或者任何你希望 shell 实现的功能。（如果你非常有雄心壮志，你可能需要修改内核以支持你所需的内核功能；xv6 不支持太多功能。）
