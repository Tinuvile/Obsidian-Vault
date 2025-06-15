# File system

文件系统的目的是组织和存储数据，它支持用户与应用程序间的数据共享，并保证数据的持久性。`xv6`文件系统提供了类似`Unix`的文件、目录和路径名，并将数据存储在`virtio`磁盘上以实现持久性。文件系统解决了几个挑战：
- 文件系统需要磁盘上的数据结构来表示命名目录和文件的树结构，记录保存每个文件内容的块的身份，并记录磁盘的哪些区域是空闲的；
- 文件系统需要支持崩溃恢复，即如果发生崩溃，在重启后它依然要能正常工作。这点的问题在于崩溃可能会中断一些更新，并留下不一致的磁盘数据结构；
- 不同的进程可能同时操作文件系统，因此文件系统代码必须协调它们以维护不变性；
- 访问磁盘比访问内存慢好几个数量级，因此文件系统必须在内存中维护一个常用块的缓存。

---

## Overview

`xv6`文件系统的实现分为七层。
1. 磁盘层在`virtio`硬件驱动器上读写块；
2. 缓冲区缓冲层缓存磁盘块并同步对它们的访问，确保一次只有一个内核进程可以修改存储在特定块中的数据；
3. 日志层允许更高层将多个块的更新包装在一个事务中，并确保在崩溃时块能原子性更新（即全部更新或全部不更新）；
4. `Inode`层提供单个文件，每个文件表示为一个具有唯一`i-number`的`inode`和一些包含文件数据的块；
5. 目录层将每个目录实现为一种特殊的`inode`，其内容是一系列目录条目，每个条目包含文件名和`i-number`；
6. 路径名层提供像`/usr/rtm/xv6/fs.c`这样的分层路径名，并通过递归查找解析它们；
7. 文件描述符层使用文件系统接口抽象了许多**Unix**资源（如管道、设备、文件等）；

![[Pasted image 20250612093153.png]]

文件系统需要有一个结构用来在磁盘上存储`inode`和内容块的位置。为此，`xv6`将磁盘划分为了几个部分。文件系统不使用块0（它包含**boot sector**）；块1叫超级块，包含有关文件系统的元数据（以块为单位的文件系统的大小、数据块的数量、`inode`的数量以及日志中的块数）；块2是日志；日志之后是`inode`，每个块中有多个`inode`；接下来是位图`bitmap`块，用于跟踪哪些数据块正在使用；其余的块是数据块，每个数据块要么在位图块中标记为空闲，要么就包含文件或目录的内容。超级块由一个名为`mkfs`的单独程序填充，该程序构建初始的文件系统。

![[Pasted image 20250612103211.png]]

---

## Buffer cache layer

缓冲区缓存有两个任务：
1. 同步对磁盘块的访问，确保内存中只有一个块的副本，并且一次只有一个内核线程使用该副本；
2. 缓存常用块。

这部分的代码位于`bio.c`，缓冲区缓存导出的主要接口有`bread`和`bwrite`。前者获取一个包含块副本的`buf`，可以在内存中读取或修改；后者将修改后的缓冲区写入磁盘上的适当块。内核线程在使用完缓冲区后必须通过调用`brelse`来释放它。缓冲区缓存使用每个缓冲区的睡眠锁来确保每次只有一个线程使用每个缓冲区和磁盘块，`bread`返回一个锁定的缓冲区，`brelse`释放锁。

```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;
```

缓冲区缓存需要有固定数量的缓冲区来保存磁盘块。如果文件系统请求一个不在缓存中的块，它需要回收当前持有的其他块的缓冲区，它会回收最近最少使用的缓冲区，这里假设的是最近最少使用的缓冲区最不可能被再次使用。

---

## Code：Buffer cache

缓冲区缓存是一个双向链表，由`main`函数调用`binit`，使用静态数组`buf`中的`NBUF`缓冲区初始化列表。所有其他对缓冲区缓存的访问都通过`bcache.head`引用链表，而不是直接访问`buf`数组。

```c
void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  // Create linked list of buffers
  bcache.head.prev = &bcache.head;
  bcache.head.next = &bcache.head;
  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    initsleeplock(&b->lock, "buffer");
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
}
```

一个缓冲区有两个与之关联的状态字段。字段`valid`表示缓冲区包含块的副本，字段`disk`表示缓冲区内容已交给磁盘，磁盘可能会更改缓冲区。`bread`调用`bget`来获取给定扇区的缓冲区。

```c
// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}
```

如果缓冲区需要从磁盘读取，`bread`在返回缓冲区前会调用`virtio_disk_rw`来完成读取操作。

```c
// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```

`bget`扫描缓冲区列表，寻找具有给定设备和扇区号的缓冲区。
- 如果存在，它会获取该缓冲区的`sleeplock`，然后返回已锁定的缓冲区。
- 而如果没有为给定扇区缓存的缓冲区，`bget`必须创建一个。它再次扫描缓冲区列表，寻找一个未使用的缓冲区（`b->refcnt = 0`）。然后编辑缓冲区元数据以记录新的设备和扇区号，并获取其睡眠锁。`b->valid = 0`确保`bread`将从磁盘读取块数据，而不是错误地使用缓冲区之前的内容。

每个磁盘扇区最多只能有一个缓存缓冲区，以确保写入操作被所有读者知道。

由于文件系统使用缓冲区上的锁进行同步，`bget`在第一个循环（检查块是否被缓存）到第二个循环（声明块现在被缓存）期间持续持有`bache.lock`来确保不变性。这样可以使`bget`中的这两个操作成为原子操作。

另外，`bget`在`bcache.lock`临界区外获取缓冲区的睡眠锁是安全的，因为非零的`b->refcnt`防止了缓冲区被重新用于不同的目的。

而如果所有缓冲区都忙，表明有太多进程在同时执行文件系统调用，`bget`会触发`panic`。一个更优雅的响应方式是休眠直到一个缓冲区变为空闲，不过这样可能触发死锁。

一旦`bread`读取了磁盘（如果需要）并将缓冲区返回给调用者，调用者就拥有缓冲区的独占使用权，可以读取或写入字节。如果调用者修改了缓冲区，它必须在释放缓冲区前调用`bwrite`将更改后的数据写入磁盘。

```c
// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}
```

`bwrite`调用`virtio_disk_rw`与磁盘硬件通信。

当调用者使用完一个缓冲区时，必须调用`brelse`来释放它。

```c
// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
  
  release(&bcache.lock);
}
```

`brelse`释放睡眠锁并将缓冲区移动到链表的前面，这样会导致链表按缓冲区最近使用的顺序排列。`bget`中的两个循环也利用到了这点，它们首先检查最近使用的缓冲区`bcache.head.next`，这样可以减少扫描时间，选择要重用的缓冲区时则向后扫描`bcache.head.prev`，选择最久未使用的缓冲区。

---

## Logging layer

文件系统设计中的一个问题是崩溃恢复，它是由于许多文件系统操作涉及对磁盘的多次写入，而在部分写入后发生的崩溃可能使磁盘上的文件系统处于不一致的状态。

`xv6`通过日志解决这个问题。
- `xv6`系统调用不会直接写入磁盘文件系统的数据结构，它将希望进行的磁盘写入操作的描述记录在磁盘的日志中。
- 一旦系统调用记录了所有的写入操作，它会在磁盘上写入一个特殊的提交记录，表示日志中包含一个完整的操作。
- 然后，系统调用将这些写入操作复制到磁盘文件系统的数据结构中。
- 写入完成后，系统调用会擦除磁盘上的日志。

如果系统崩溃并重启，在运行任何进程前，文件系统代码会按以下方式恢复：
- 如果日志标记为包含完整操作，则恢复代码并将写操作复制到磁盘文件系统中它们应属的位置。
- 如果日志未标记为包含完整操作，则恢复代码将忽略该日志，通过擦除日志来完成操作。

日志使得操作相对于崩溃是原子的。

---

## Log design

日志位于超级块中指定的已知固定位置，它由一个头部块和一系列更新的块副本（日志块）组成。

```c
// Contents of the header block, used for both the on-disk header block
// and to keep track in memory of logged block# before commit.
struct logheader {
  int n;
  int block[LOGSIZE];
};
```

头部块包含一个扇区号数组，每个日志块对应一个扇区号，以及日志块的数量。磁盘上头部块中的计数为0表示日志中没有事务，其他表示日志中包含一个完整的已提交事务