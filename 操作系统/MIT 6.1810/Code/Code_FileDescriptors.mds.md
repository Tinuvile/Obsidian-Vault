# FileDescriptors相关源码阅读

---

涉及的文件主要有`file.h`和`sysfile.c`。

---

## file.h

`file`结构体表示系统中的一个打开文件实例：

```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE, FD_DEVICE } type;
  int ref; // reference count
  char readable;
  char writable;
  struct pipe *pipe; // FD_PIPE
  struct inode *ip;  // FD_INODE and FD_DEVICE
  uint off;          // FD_INODE
  short major;       // FD_DEVICE
};
```

`type`区分类型，`ref`用于跟踪有多少个文件描述符引用该结构，后面两个读写标志位，`off`为偏移量。

`inode`结构为磁盘`inode`的内存缓存副本：

```c
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

---

## sysfile.c

这段代码中只看`fdalloc`函数，这个函数是文件描述符的分配函数。在此之前，先看一下`proc.h`中的定义：

```c
// Per-process state
struct proc {
  struct file *ofile[NOFILE];  // Open files
};
```

再回来看这个函数：

```c
// Allocate a file descriptor for the given file.
// Takes over file reference from caller on success.
static int
fdalloc(struct file *f)
{
  int fd;
  struct proc *p = myproc();

  for(fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd] == 0){
      p->ofile[fd] = f;
      return fd;
    }
  }
  return -1;
}
```

它首先获取当前进程，然后遍历当前进程的文件描述符，`NOFILE`定义在`param.h`中：

```c
#define NOFILE       16  // open files per process
```

如果发现空槽位，则进行分配。
































