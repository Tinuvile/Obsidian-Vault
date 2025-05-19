> There is a saying in computer systems that any systems problem can be solved with a level of indirection.

这个Lab是实现`copy-on-write`。

---

## 回顾

`COW fork()`仅为子进程创建一个页表，其中的用户内存页表项PTE指向父进程的物理页。`COW fork()`会将父进程和子进程中所有用户 PTE 标记为只读，当任一进程尝试写入这些 COW 页时， CPU 将触发`page fault`进行处理，为进程分配一个新的物理页，复制原始内容到新页中，并修改进程中的相关 PTE 指向新页，标记为可写。

需要注意的是这种机制会使一个物理页可能被多个进程的页表引用，在释放物理页时需要注意。

---

## Implement copy-on-write fork([hard](https://pdos.csail.mit.edu/6.S081/2024/labs/guidance.html))

> Your task is to implement copy-on-write fork in the xv6 kernel. You are done if your modified kernel executes both the cowtest and 'usertests -q' programs successfully.  
> 你的任务是在 xv6 内核中实现写时复制（copy-on-write）的 fork 功能。当你修改后的内核能成功执行 cowtest 和'usertests -q'程序时，即告完成。

还是先看一下测试的代码`cowtest.c`。测试分为`simpletest`、`threetest`、`filetest`和`forkforktest`四块。

`simpletest`会分配超过一半的可用物理内存，然后调用`fork()`。在没有实现的情况下，由于没有足够的空闲物理内存为子进程，`fork()`操作会失败。

```c
$ cowtest
simple: fork() failed
```

`threetest`创建三个嵌套的进程，并让它们同时修改 COW 页。检查每个进程看到的页面内容是否独立以及测试内存回收机制。

`filetest`通过四次循环创建多个管道和子进程，测试 COW 机制下通过管道进行进程间通信。

`forkforktest`是压力测试，创建大量进程。

> 确保当最后一个页表项（PTE）引用消失时释放对应的物理页——但不得提前释放。实现这一点的有效方法是为每个物理页维护一个"引用计数"，记录有多少用户页表引用了该页。当`kalloc()`分配页面时，将其引用计数设为 1。当`fork`操作导致子进程共享页面时递增该计数，每当任何进程从其页表中移除该页面时递减计数。仅当页面引用计数为零时，`kfree()`才应将其放回空闲列表。可以使用固定大小的整数数组来存储这些计数。你需要设计数组的索引方案并确定其大小。例如，可以用页面的物理地址除以 4096 作为数组索引，数组大小则设为`kalloc.c`中`kinit()`放入空闲列表的页面最高物理地址对应的元素数量。可自由修改`kalloc.c`（如`kalloc()`和`kfree()`）以维护引用计数。

先完成这一部分，在`kmem`中添加一个数组来记录每个页的引用计数。

```c
struct {
  struct spinlock lock;
  struct run *freelist;
  uint16 ref_count[(PHYSTOP - KERNBASE) / PGSIZE];
} kmem;
```

另外，在`riscv.h`中添加物理地址转化为索引的宏定义。

```c
#define PA2INDEX(pa) (((uint64)(pa) - KERNBASE) >> PGSHIFT)
```

系统启动时的初始化在`freerange`中实现，设置初始值为1，然后调用`kfree`递减到0。

```c
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE) {
    kmem.ref_count[PA2INDEX(p)] = 1;
    kfree(p);
  }
}
```

分配页面时的初始化则在`kalloc`中，将该页的引用计数设置为1。

```c
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
    kmem.freelist = r->next;
    kmem.ref_count[PA2INDEX(r)] = 1;
  }
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

`kfree`负责在页面引用计数降到0时释放。

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&kmem.lock);

  uint64 index = PA2INDEX(pa);
  if (kmem.ref_count[index] == 0)
    panic("kfree ref_count");

  if (--kmem.ref_count[index] == 0) {
    r = (struct run*)pa;
    memset(pa, 1, PGSIZE); // fill with junk
    r->next = kmem.freelist;
    kmem.freelist = r;
  }

  release(&kmem.lock);
}
```

> 修改`uvmcopy()`，将父进程的物理页面映射到子进程，而非分配新页面。对于设置了 PTE_W 标志的页面，需清除父子进程 PTE 中的 PTE_W 标志。

这里如果设置了 PTE_W，会把它清除，并加入 PTE_COW 标志。

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if (flags & PTE_W) {
      flags &= ~PTE_W;
      flags |= PTE_COW;
    }
    *pte = PA2PTE(pa) | flags;
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0) {
      goto err;
    }
    krefer((void*)pa);
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

然后还要修改引用计数，在`kalloc.c`中加一个函数`krefer`添加引用计数。

```c
void
krefer(void *pa)
{
  if(((uint64)pa % PGSIZE)!= 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kerfer");

  acquire(&kmem.lock);

  if (kmem.ref_count[PA2INDEX(pa)] == 0)
    panic("kerfer ref_count");

  kmem.ref_count[PA2INDEX(pa)]++;

  release(&kmem.lock);
}
```

> 修改`usertrap()`以识别页错误。当可写 COW 页发生写入页错误时，使用`kalloc()`分配新页，复制旧页内容到新页，并在 PTE 中安装新页且设置 PTE_W 标志。原本只读的页（如代码段未映射 PTE_W 的页）应保持只读并在父子进程间共享；尝试写入此类页的进程应被终止。

在`usertrap`中加入写入页错误的情况的特殊处理。

![](C:\Users\ASUS\AppData\Roaming\marktext\images\2025-04-30-19-05-56-image.png)

如果错误与预期不符就直接杀掉进程。

```c
    else if(r_scause() == 15){
    // store page fault
    if (uvmcow(p->pagetable, r_stval(), 1) < 0)
      setkilled(p);
  }
```

`uvmcow`负责具体的处理。调用`kalloc`新分配一个页，并清除 PTE_COW 标志，重新设置 PTE_W 标志。然后复制内容并映射。

```c
int
uvmcow(pagetable_t pagetable, uint64 va, uint64 npages)
{
  if (va >= MAXVA)
    return -1;

  va = PGROUNDDOWN(va);

  for (uint64 a = va; a < va + npages * PGSIZE; a += PGSIZE) {
    pte_t *pte = walk(pagetable, a, 0);
    if (pte == 0)
      goto err;
    
    int flags = PTE_FLAGS(*pte);

    if ((flags & PTE_V) == 0 || (flags & PTE_U) == 0 || (flags & PTE_COW) == 0)
      goto err;

    flags &= ~PTE_COW;
    flags |= PTE_W;
    
    void *mem = kalloc();
    if (mem == 0)
      goto err;

    memmove(mem, (void *)PTE2PA(*pte), PGSIZE);

    uvmunmap(pagetable, va, 1, 1);

    if (mappages(pagetable, a, PGSIZE, (uint64)mem, flags) < 0)
      goto err;
  }
  return 0;

err:
  return -1;
}
```

> 修改 copyout()函数，使其在遇到写时复制（COW）页时采用与缺页异常相同的处理机制。

```c
    if(pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0)
      return -1;
    if ((*pte & PTE_COW) != 0)
      uvmcow(pagetable, va0, 1);
    if ((*pte & PTE_W) == 0)
      return -1;
```

运行测试：

```bash
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ cowtest
simple: ok
simple: ok
three: ok
three: ok
three: ok
file: ok
forkfork: ok
ALL COW TESTS PASSED
```

---

## Optional challenge exercise

> Measure how much your COW implementation reduces the number of bytes xv6 copies and the number of physical pages it allocates. Find and exploit opportunities to further reduce those numbers.  
> 测量你的写时复制(COW)实现减少了多少 xv6 复制的字节数和分配的物理页数。寻找并利用进一步减少这些数值的机会。




































