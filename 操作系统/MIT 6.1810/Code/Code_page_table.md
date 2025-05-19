# page table源码阅读

---

相关的代码文件主要是[`kalloc.c`]([xv6-riscv/kernel/kalloc.c at riscv · mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kalloc.c))、[`vm.c`]([xv6-riscv/kernel/vm.c at riscv · mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c))、[`memlayout.h`]([xv6-riscv/kernel/memlayout.h at riscv · mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/memlayout.h))和[`riscv.h`]([xv6-riscv/kernel/riscv.h at riscv · mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/riscv.h))。

---

## memlayout.h

这是`xv6`的物理内存布局定义头文件。首先注释说明的是QEMU虚拟机的物理内存布局。

```c
// Physical memory layout

// qemu -machine virt is set up like this,
// based on qemu's hw/riscv/virt.c:
//
// 00001000 -- boot ROM, provided by qemu
// 02000000 -- CLINT
// 0C000000 -- PLIC
// 10000000 -- uart0 
// 10001000 -- virtio disk 
// 80000000 -- boot ROM jumps here in machine mode
//             -kernel loads the kernel here
// unused RAM after 80000000.

// the kernel uses physical memory thus:
// 80000000 -- entry.S, then kernel text and data
// end -- start of kernel page allocation area
// PHYSTOP -- end RAM used by the kernel
```

当机器启动时，首先执行`entry.S`中的汇编启动代码，这是内核的入口，`0x80000000`是内核启动的物理地址，内核的代码和数据在这里存放。

`end`是内核代码和数据的结束地址，这个地址之后是内核动态分配内存的区域。

### PHYSTOP

`PHYSTOP`是内核可使用的物理内存上限。

```c
// the kernel expects there to be RAM
// for use by the kernel and user pages
// from physical address 0x80000000 to PHYSTOP.
#define KERNBASE 0x80000000L
#define PHYSTOP (KERNBASE + 128*1024*1024)
```

这一段定义了内核代码起始地址和结束地址，DRAM的物理内存上限为128MB。

```tex
0x80000000 ┌───────────────┐
           │ 内核代码/数据   │ ← 约占用几MB
           ├───────────────┤ ← end符号
           │ 内核页表       │ 
           ├───────────────┤
           │ 用户进程页表    │ 
           ├───────────────┤
           │ 用户进程数据页  │ 
           ├───────────────┤
           │ 动态分配内存    │ 
0x88000000 └───────────────┘ ← PHYSTOP（128MB）
```

### TRAMPOLINE

接下来是**TRAMPOLINE**页，它被映射到虚拟地址空间的最高处，在用户空间与内核空间都一样，它的内容在`trampoline.S`中设置。

```c
// map the trampoline page to the highest address,
// in both user and kernel space.
#define TRAMPOLINE (MAXVA - PGSIZE)
```

### KSTACK

**KSTACK**宏定义用于计算每个进程的内核栈虚拟地址。

内核栈位于`trampoline`下面，每个内核栈又由实际栈空间和上下保护页组成，保护页有共有部分。

KSTACK计算出的地址是从下保护页开始的，实际栈空间要从`KSTACK(p)+PGSIZE`开始。

```c
// map kernel stacks beneath the trampoline,
// each surrounded by invalid guard pages.
#define KSTACK(p) (TRAMPOLINE - (p)*2*PGSIZE - 3*PGSIZE)
```

### TRAMPOLINE

### USYSCALL

下面这段注释介绍了用户进程虚拟地址空间的布局。**TRAMPOLINE**下面是**TRAPFRAME**陷阱帧，然后是**USYSCALL**系统调用页，包含进程ID信息。

```c
// User memory layout.
// Address zero first:
//   text
//   original data and bss
//   fixed-size stack
//   expandable heap
//   ...
//   USYSCALL (shared with kernel)
//   TRAPFRAME (p->trapframe, used by the trampoline)
//   TRAMPOLINE (the same page as in the kernel)
#define TRAPFRAME (TRAMPOLINE - PGSIZE)
#ifdef LAB_PGTBL
#define USYSCALL (TRAPFRAME - PGSIZE)

struct usyscall {
  int pid;  // Process ID
};
#endif
```

---

## riscv.h

这个文件中有很多用于硬件寄存器访问的内联函数，就先跳过。我们主要关注一些宏定义。

### MSTATUS

```c
// Machine Status Register, mstatus

#define MSTATUS_MPP_MASK (3L << 11) // previous mode.
#define MSTATUS_MPP_M (3L << 11)
#define MSTATUS_MPP_S (1L << 11)
#define MSTATUS_MPP_U (0L << 11)
#define MSTATUS_MIE (1L << 3)    // machine-mode interrupt enable.
```

这是RISC-V的机器状态寄存器`mstatus`相关的宏。先解释一下后面的`3L << 11`的含义：

`3L`即为二进制的`11`，`<< 11`意为左移11位，结果是`0b1100000000000`，对应`mstatus`寄存器的11-12位（寄存器位从0开始），这两位是寄存器中的MMP字段（Machine Previous Privilege）。

- `MSTATUS_MPP_MASK`用于屏蔽`mstatus`中的MMP字段，它会覆盖11和12位；

- `MSTATUS_MPP_M`表示进入异常前特权级为机器模式，其11和12位均为1；

- `MSTATUS_MPP_S`表示进入异常前特权级为监督模式，其11位为1，12位为0；

- `MSTATUS_MPP_U`表示进入异常前特权级为用户模式，其11和12位均为0。

`MSTATUS_MIE`在第3位，`MIE=1`时允许机器模式下的中断，`MIE=0`时允许所有机器模式中断。

### SSTATUS

```c
// Supervisor Status Register, sstatus

#define SSTATUS_SPP (1L << 8)  // Previous mode, 1=Supervisor, 0=User
#define SSTATUS_SPIE (1L << 5) // Supervisor Previous Interrupt Enable
#define SSTATUS_UPIE (1L << 4) // User Previous Interrupt Enable
#define SSTATUS_SIE (1L << 1)  // Supervisor Interrupt Enable
#define SSTATUS_UIE (1L << 0)  // User Interrupt Enable
```

**SPP**标识异常发生前的特权模式；**SPIE**保存异常前监督模式的中断使能状态，异常发生时，硬件会将当前**SIE**的值保存到这里，然后禁用中断；**UPIE**保存异常前用户模式的中断使能状态，与前面同理；**SIE**和**UIE**则是控制各模式中断使能的。

### SIE

```c
// Supervisor Interrupt Enable
#define SIE_SEIE (1L << 9) // external
#define SIE_STIE (1L << 5) // timer
#define SIE_SSIE (1L << 1) // software
```

这三个宏用于控制监督模式下不同类型中断的使能开关。分别控制外部中断、定时器中断和软件中断。

### SATP

```c
// use riscv's sv39 page table scheme.
#define SATP_SV39 (8L << 60)

#define MAKE_SATP(pagetable) (SATP_SV39 | (((uint64)pagetable) >> 12))
```

这两句用于配置SATP寄存器。

第一句设置其分页模式为SV-39，`1000`是MODE字段的SV39编码，用来指明分页模式。

SATP寄存器的结构如下。ASID是地址空间标识符；PPN就是根页表的物理页号。

```c
63      60        59              44              0
+-------+----------+-------------------------------+
| 1000  |  ASID    |          PPN (44 bits)        |
+-------+----------+-------------------------------+
```

第二句用于构造SATP寄存器值，将根页表的物理地址转换为符合SATP寄存器格式的值。`SATP_SV39`设置分页模式；`(((uint64)pagetable) >> 12`完成物理地址到PPN的转换，物理地址需按4KB对齐，右移12位也就相当于取高44位作为PPN。

> RISC-V的Sv39分页模式下，物理地址根据规范最大支持56位。物理地址由PPN和页内偏移组成。

### pte_t和pagetable_t

```c
typedef uint64 pte_t;
typedef uint64 *pagetable_t; // 512 PTEs
```

`pte_t`对应页表项的64位数据结构、`pagetable_t`则是页表指针。

### PG宏

```c
#define PGSIZE 4096 // bytes per page
#define PGSHIFT 12  // bits of offset within a page
```

分别定义了页的大小以及页偏移量（用于地址转换）。

```c
#ifdef LAB_PGTBL
#define SUPERPGSIZE (2 * (1 << 20)) // bytes per page
#define SUPERPGROUNDUP(sz)  (((sz)+SUPERPGSIZE-1) & ~(SUPERPGSIZE-1))
#endif
```

定义超级页的大小为2MB，`SUPERPGROUNDUP`是地址对齐宏。

```c
#define PGROUNDUP(sz)  (((sz)+PGSIZE-1) & ~(PGSIZE-1))
#define PGROUNDDOWN(a) (((a)) & ~(PGSIZE-1))
```

这两个是普通页的地址对齐宏，`PGROUNDUP`用于向上对齐，`PGROUNDDOWN`向下对齐。

### PTE

```c
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // user can access
```

这定义的是PTE中的标志位。

再下面就是跟页表处理相关的一些宏定义。

```c
#if defined(LAB_MMAP) || defined(LAB_PGTBL)
#define PTE_LEAF(pte) (((pte) & PTE_R) | ((pte) & PTE_W) | ((pte) & PTE_X))
#endif
```

`PTE_LEAF`判断是否为叶子页表项，通过检查是否具有`R/W/X`任一权限。

```c
// shift a physical address to the right place for a PTE.
#define PA2PTE(pa) ((((uint64)pa) >> 12) << 10)

#define PTE2PA(pte) (((pte) >> 10) << 12)

#define PTE_FLAGS(pte) ((pte) & 0x3FF)
```

`PA2PTE`将物理地址转换为页表项，先取物理地址的高44位，再右移保留10位作为标志位。`PTE2PA`同理。`PTE_FLAGS`提取PTE标志位。

### PX宏

```c
// extract the three 9-bit page table indices from a virtual address.
#define PXMASK          0x1FF // 9 bits
#define PXSHIFT(level)  (PGSHIFT+(9*(level)))
#define PX(level, va) ((((uint64) (va)) >> PXSHIFT(level)) & PXMASK)
```

这是关于虚拟地址的宏定义。`PXMASK`是9位掩码，用于提取低9位；`PXSHIFT`用于计算各级页表的位偏移量；`PX`提取指定层级的页表索引。

### MAXVA

```c
// one beyond the highest possible virtual address.
// MAXVA is actually one bit less than the max allowed by
// Sv39, to avoid having to sign-extend virtual addresses
// that have the high bit set.
#define MAXVA (1L << (9 + 9 + 9 + 12 - 1))
```

最后是最大虚拟地址。

---

## kalloc.c

这部分代码主要关于`xv6`系统的物理内存管理。

```c
extern char end[]; // first address after kernel.
                   // defined by kernel.ld.
```

这是`kernel`的结束地址。

### struct

```c
struct run { // 空闲页链表节点
  struct run *next;
};

struct { // 内存管理全局状态
  struct spinlock lock; // 多核同步锁
  struct run *freelist; // 空闲页链表
} kmem;
```

### kinit

`kinit()`初始化内存分配器的自旋锁，然后将内核结束后的物理内存加入空闲链表。

```c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```

### freerange

其中调用的`freerange()`函数用于将指定范围内的物理内存页初始化为空闲状态。

```c
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}
```

### kfree

`kfree()`用于释放指定的物理页，安全检查包括是否对齐、是否在内核页中、以及是否超出物理内存上限。`memset`填充垃圾数据，然后将其加入空闲链表（头插法）。

```c
// Free the page of physical memory pointed at by pa,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```

### kalloc

`kalloc()`用于分配物理页。它会取空闲链表头部指向的物理页，填充垃圾数据后返回该页指针。

```c
// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

---

## vm.c

这部分代码主要关于`xv6`的虚拟内存管理。

### kvminit

```c
/*
 * the kernel's page table.
 */
pagetable_t kernel_pagetable;

extern char etext[];  // kernel.ld sets this to end of kernel code.

extern char trampoline[]; // trampoline.S

// Initialize the one kernel_pagetable
void
kvminit(void)
{
  kernel_pagetable = kvmmake();
}
```

`kernel_pagetable`是内核页表指针。

### kvmmake

```c
// Make a direct-map page table for the kernel.
pagetable_t
kvmmake(void)
{
  pagetable_t kpgtbl;  // 内核页表指针

  kpgtbl = (pagetable_t) kalloc();  // 分配页表空间
  memset(kpgtbl, 0, PGSIZE);  // 填充

  // uart registers
  kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(kpgtbl, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

#ifdef LAB_NET
  // PCI-E ECAM (configuration space), for pci.c
  kvmmap(kpgtbl, 0x30000000L, 0x30000000L, 0x10000000, PTE_R | PTE_W);

  // pci.c maps the e1000's registers here.
  kvmmap(kpgtbl, 0x40000000L, 0x40000000L, 0x20000, PTE_R | PTE_W);
#endif  

  // PLIC
  kvmmap(kpgtbl, PLIC, PLIC, 0x4000000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(kpgtbl, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap(kpgtbl, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);

  // allocate and map a kernel stack for each process.
  proc_mapstacks(kpgtbl);

  return kpgtbl;
}
```

`kvmmake()`用于创建内核页表，这个页表是直接映射的。

### kvminithart

```c
// Switch h/w page table register to the kernel's page table,
// and enable paging.
void
kvminithart()
{
  // wait for any previous writes to the page table memory to finish.
  sfence_vma();

  w_satp(MAKE_SATP(kernel_pagetable));

  // flush stale entries from the TLB.
  sfence_vma();
}
```

这是激活内核页表的函数，用于完成MMU（内存管理单元）的页表切换操作，启用分页并将SATP指向内核页表。

### walk

```c
// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
#ifdef LAB_PGTBL
      if(PTE_LEAF(*pte)) {
        return pte;
      }
#endif
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

`walk()`用于查找虚拟地址对应的页表项。`PX`查找页表项的索引，在前面的头文件中定义的。如果PTE有效，就用`PTE2PA`转化为物理地址并赋值给`pagetable`，来指向下一级的页表。后面的`#ifdef LAB_PGTBL`是大页情况，无需再遍历下级页表。`else`处理的是PTE无效，需要分配的情况。最后返回最终层的PTE指针。

### walkaddr

```c
// Look up a virtual address, return the physical address,
// or 0 if not mapped.
// Can only be used to look up user pages.
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}
```

`walkaddr()`用于查找虚拟地址对应的物理地址。

### kvmmap

```c
// add a mapping to the kernel page table.
// only used when booting.
// does not flush TLB or enable paging.
void
kvmmap(pagetable_t kpgtbl, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(kpgtbl, va, sz, pa, perm) != 0)
    panic("kvmmap");
}
```

`kvmmap()`用于内核页表映射。

### mappages

```c
// Create PTEs for virtual addresses starting at va that refer to
// physical addresses starting at pa.
// va and size MUST be page-aligned.
// Returns 0 on success, -1 if walk() couldn't
// allocate a needed page-table page.
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("mappages: va not aligned");

  if((size % PGSIZE) != 0)
    panic("mappages: size not aligned");

  if(size == 0)
    panic("mappages: size");

  a = va;
  last = va + size - PGSIZE;
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("mappages: remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```

`mappages()`用于将虚拟地址范围 `[va, va+size)` 映射到物理地址范围 `[pa, pa+size)`，并设置页表项（PTE）的权限。首先获取当前虚拟地址对应的PTE，然后检查是否已经被占用，再设置PTE的物理地址（`PA2PTE`）和权限。

### uvmunmap

```c
// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory.
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;
  int sz;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += sz){
    sz = PGSIZE;
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0) {
      printf("va=%ld pte=%ld\n", a, *pte);
      panic("uvmunmap: not mapped");
    }
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

`uvmunmap()`用于解除虚拟内存映射，`do_free`决定是否释放物理页。

### uvmcreate

```c
// create an empty user page table.
// returns 0 if out of memory.
pagetable_t
uvmcreate()
{
  pagetable_t pagetable;
  pagetable = (pagetable_t) kalloc();
  if(pagetable == 0)
    return 0;
  memset(pagetable, 0, PGSIZE);
  return pagetable;
}
```

`uvmcreate()`用于创建空的用户页表。

### uvmfirst

```c
// Load the user initcode into address 0 of pagetable,
// for the very first process.
// sz must be less than a page.
void
uvmfirst(pagetable_t pagetable, uchar *src, uint sz)
{
  char *mem;

  if(sz >= PGSIZE)
    panic("uvmfirst: more than a page");
  mem = kalloc();
  memset(mem, 0, PGSIZE);
  mappages(pagetable, 0, PGSIZE, (uint64)mem, PTE_W|PTE_R|PTE_X|PTE_U);
  memmove(mem, src, sz);
}
```

`uvmfirst()`初始化第一个用户进程的地址空间，将初始化代码映射到虚拟地址`0`。

### uvmalloc

```c
// Allocate PTEs and physical memory to grow process from oldsz to
// newsz, which need not be page aligned.  Returns new size or 0 on error.
uint64
uvmalloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz, int xperm)
{
  char *mem;
  uint64 a;
  int sz;

  if(newsz < oldsz)
    return oldsz;

  oldsz = PGROUNDUP(oldsz);
  for(a = oldsz; a < newsz; a += sz){
    sz = PGSIZE;
    mem = kalloc();
    if(mem == 0){
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
#ifndef LAB_SYSCALL
    memset(mem, 0, sz);
#endif
    if(mappages(pagetable, a, sz, (uint64)mem, PTE_R|PTE_U|xperm) != 0){
      kfree(mem);
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
  }
  return newsz;
}
```

`uvmalloc()`用于扩展用户进程的内存空间。

### uvmdealloc

```c
// Deallocate user pages to bring the process size from oldsz to
// newsz.  oldsz and newsz need not be page-aligned, nor does newsz
// need to be less than oldsz.  oldsz can be larger than the actual
// process size.  Returns the new process size.
uint64
uvmdealloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  if(newsz >= oldsz)
    return oldsz;

  if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 1);
  }

  return newsz;
}
```

`uvmdealloc()`缩小用户进程的内存空间。

### freewalk

```c
// Recursively free page-table pages.
// All leaf mappings must already have been removed.
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}
```

`freewalk()`用于递归释放页表页，在进程退出时彻底释放页表占用的所有物理内存。

### uvmfree

```c
// Free user memory pages,
// then free page-table pages.
void
uvmfree(pagetable_t pagetable, uint64 sz)
{
  if(sz > 0)
    uvmunmap(pagetable, 0, PGROUNDUP(sz)/PGSIZE, 1);
  freewalk(pagetable);
}
```

`uvmfree()`用于释放用户内存，先接触映射，再递归释放页表页占用的物理内存。

### uvmcopy

```c
// Given a parent process's page table, copy
// its memory into a child's page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;
  int szinc;

  for(i = 0; i < sz; i += szinc){
    szinc = PGSIZE;
    szinc = PGSIZE;
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

`uvmcopy()`用于将父进程的页表和物理内存复制给子进程。在使用`pa`和`flags`获取父进程信息后，直接用`memmove`把内容复制给`kalloc`新分配的子进程物理页，然后用`mappages`映射。

### uvmclear

```c
// mark a PTE invalid for user access.
// used by exec for the user stack guard page.
void
uvmclear(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    panic("uvmclear");
  *pte &= ~PTE_U;
}
```

`uvmclear()`取消用户模式的访问权限。

### copy

```c
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  pte_t *pte;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if (va0 >= MAXVA)
      return -1;
    if((pte = walk(pagetable, va0, 0)) == 0) {
      // printf("copyout: pte should exist 0x%x %d\n", dstva, len);
      return -1;
    }


    // forbid copyout over read-only user text pages.
    if((*pte & PTE_W) == 0)
      return -1;

    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}

// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}

// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  uint64 n, va0, pa0;
  int got_null = 0;

  while(got_null == 0 && max > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > max)
      n = max;

    char *p = (char *) (pa0 + (srcva - va0));
    while(n > 0){
      if(*p == '\0'){
        *dst = '\0';
        got_null = 1;
        break;
      } else {
        *dst = *p;
      }
      --n;
      --max;
      p++;
      dst++;
    }

    srcva = va0 + PGSIZE;
  }
  if(got_null){
    return 0;
  } else {
    return -1;
  }
}
```

`copyout`函数用于从内核中复制数据到用户空间，`copyin`相反，`copystr`用于从用户空间复制字符串到内核。
