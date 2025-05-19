# exec源码阅读

---

## loadseg

这是内核中用于加载程序段的函数，它负责将程序段加载到页表的虚拟地址`va`处。

```c
// Load a program segment into pagetable at virtual address va.
// va must be page-aligned
// and the pages from va to va+sz must already be mapped.
// Returns 0 on success, -1 on failure.
static int
loadseg(pagetable_t pagetable, uint64 va, struct inode *ip, uint offset, uint sz)
{
  uint i, n;
  uint64 pa;

  for(i = 0; i < sz; i += PGSIZE){
    pa = walkaddr(pagetable, va + i);
    if(pa == 0)
      panic("loadseg: address should exist");
    if(sz - i < PGSIZE)
      n = sz - i;
    else
      n = PGSIZE;
    if(readi(ip, 0, (uint64)pa, offset+i, n) != n)
      return -1;
  }

  return 0;
}
```

其中`readi`用于从文件读取内容到物理内存中。

---

## flags2perm

这个函数负责将ELF程序头标志转换为页表权限位。

```c
int flags2perm(int flags)
{
    int perm = 0;
    if(flags & 0x1)
      perm = PTE_X;
    if(flags & 0x2)
      perm |= PTE_W;
    return perm;
}
```

---

## exec

> `begin_op`和`end_op`是`xv6`系统日志相关的函数。

```c
if((ip = namei(path)) == 0){
  end_op();
  return -1;
} 
ilock(ip);
```

这部分通过路径查找文件`inode`，找不到即释放日志锁并返回错误码-1。`ilock`对`inode`加锁确保独占访问该文件。

```c
// Check ELF header
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;

  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;
```

然后是ELF头检查部分和页表初始化。`readi`从文件读取ELF头结构，并检查`elf_magic`。然后为进程创建新页表（替换原有页表）。

```c
  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz, flags2perm(ph.flags))) == 0)
      goto bad;
    sz = sz1;
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;
```

这部分加载ELF程序到内存，遍历程序头表，里面包含一些有效性检查，然后调用`uvmalloc`进行内存分配与映射，并用`loadseg`加载内容。最后释放`inode`锁，提交文件系统事务，`ip=0`标记`inode`已处理完成。

```c
p = myproc();
uint64 oldsz = p->sz;
```

这部分保存旧内存大小，用于后面内存替换并安全释放旧内存。

```c
  // Allocate some pages at the next page boundary.
  // Make the first inaccessible as a stack guard.
  // Use the rest as the user stack.
  sz = PGROUNDUP(sz);
  uint64 sz1;
  if((sz1 = uvmalloc(pagetable, sz, sz + (USERSTACK+1)*PGSIZE, PTE_W)) == 0)
    goto bad;
  sz = sz1;
  uvmclear(pagetable, sz-(USERSTACK+1)*PGSIZE);
  sp = sz;
  stackbase = sp - USERSTACK*PGSIZE;
```

这部分为进程分配用户栈，`+1`用于设置保护页，`uvmclear`清除保护页的`PTE_U`标志，`sp`是栈顶指针，`stackbase`是栈基地址。

```c
  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 16; // riscv sp must be 16-byte aligned
    if(sp < stackbase)
      goto bad;
    if(copyout(pagetable, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    ustack[argc] = sp;
  }
  ustack[argc] = 0;
```

这部分将命令行参数压入用户栈，并记录地址。`copyout`将参数字符串复制到用户栈，`ustack[argc]`记录参数地址，以`0`结束。

```c
  // push the array of argv[] pointers.
  sp -= (argc+1) * sizeof(uint64);
  sp -= sp % 16;
  if(sp < stackbase)
    goto bad;
  if(copyout(pagetable, sp, (char *)ustack, (argc+1)*sizeof(uint64)) < 0)
    goto bad;
```

这部分将参数指针的地址数组复制到用户栈。

```c
  // arguments to user main(argc, argv)
  // argc is returned via the system call return
  // value, which goes in a0.
  p->trapframe->a1 = sp;
```

这里修改进程的`trapframe`，当进程恢复执行时，寄存器将携带正确参数。`a1`寄存器存储参数指针数组地址。

```c
  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));
```

这部分从文件路径中提取程序名，并保存到进程结构体中，用于Debug。

那么至此，准备工作就全部完成了。

```c
  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp; // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);
```

这部分提交新的用户镜像并清理旧资源。先保存旧页表地址，然后切换至新页表并进行一些参数更新，最后释放旧页表。

```c
return argc; // this ends up in a0, the first argument to main(argc, argv)
```

返回参数给用户空间。

```c
bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
```

最后是错误处理逻辑。
