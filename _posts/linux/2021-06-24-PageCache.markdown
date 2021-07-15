---
layout:     post
title:      "PageCache"
date:       2021-06-24 11:12:00
categories: 技术
tags: Linux
---

开篇诗两句：

​        **沧海月明珠有泪，蓝田日暖玉生烟。**

​        **此情可待成追忆，只是当时已惘然。**         ——李商隐《锦瑟》



**缓存：**利用部分系统物理内存，确保最重要、最常使用的块设备数据在操作时可直接从主内存获取，而无须从低速设备读取。缓存也存在负面效应：减少了实际上可用的物理内存容量；增加了数据丢失的风险，如果系统崩溃(例如，由于停电)，缓存包含的数据可能没有写回到底层的块设备，造成不可恢复的数据丢失。

**页缓存Page Cache：**是一种软件机制，针对以页为单位的所有操作，负责了块设备的大部分缓存工作。块缓存Buffer Cache：以块为操作单位，块长度取决于特定的文件系统或其设置，一般比较小。在2.4之前的内核，一个磁盘块可以同时存于两个缓存中，这样导致必须同步操作两个缓存区，而且浪费了内存。现在独立的磁盘块通过块I/O缓冲，也要被存入页高速缓存。通过缓存，磁盘块映射到它们相关的内存页，并缓存到高速缓存中。

文件操作涉及Page Cache、虚拟文件系统VFS和块设备的处理。

## PageCache

文件系统的最小逻辑寻址单元是块，内存的最小寻址单元是页，页是内存管理的基本单位。

页缓存是用来保存磁盘中数据的**内存页**，一般大小为4KB: (1) 以页为单位的文件视图 (2) 属于某文件的缓存页以特定数据结构形式[基树数或xarray]组织，对于基数树，可以从文件inode的address_space对象获取树根，叶子节点对应文件页，从左到右排列，根据index（文件内的页索引）在基数树中查找。

页缓存作用：加速数据I/O，写数据时首先写到缓存，将写入的页标记为dirty，然后刷新到外部存储(磁盘)，即缓存写机制中的write-back；读数据时首先读取缓存，如果未命中，再去外部存储读取，并且将读取来的数据也加入缓存。

页缓存利用了“时间局部性”原理，即认为刚被访问的资源在不久后再次被访问的概率很高。在第一次访问时对数据进行缓存，避免了后续代价很高的磁盘访问。利用“空间局部性”，即认为数据往往是连续访问的。页缓存采用预读技术，每次读请求时，从磁盘中读取更多的数据到页缓存中。

系统的物理内存被划分成许多大小相同的页，一部分在操作系统启动时分配给内核使用，并常驻内存;另一部分是内核通过slab分配器动态分配的内存，用来管理进程、文件描述符等；进程使用的内存，分配给代码、数据和堆栈等；剩余的部分作为页缓存。页缓存的大小是在一直动态变化的。当系统内存充足时，页缓存会一直增大，缓存尽可能多的数据；当系统空闲内存不足时，这时如果有进程申请内存，操作系统会从Page Cache中回收内存页进行分配，如果Page Cache也已不足，那么系统会将当期驻留在内存中的数据置换到事先配置在磁盘上的交换空间中，然后空出来的这部分内存就可以用来分配了。

address_space对象是与内核中Page Cache相关的核心数据结构。每个文件(inode)对应着一个address_space对象，Page Cache的属于同一个文件的多个页链接到同一个address_space对象，address_space对应一个基数树，用于实现页的快速检索。

address_space_operations对象保存了address_space的特定操作。

```c
struct address_space {                          // 部分成员
    struct inode            *host;              // 所有者: inode
    struct radix_tree_root  page_tree;          // 所有页的基数树
    struct prio_tree_root   i_mmap;             // 树的根节点，映射的所有页都可以在树中找到
    unsigned long           nrpages;            // 页的总数
    pgoff_t                 writeback_index;    // 回写由此开始
    struct address_space_operations *a_ops;     // addrress_space可执行的所有操作
};
struct address_space_operations {
    int (*writepage)(struct page *page, struct writeback_control *wbc); 
    int (*readpage)(struct file *, struct page *); 
    int (*sync_page)(struct page *); 
    int (*writepages)(struct address_space *, struct writeback_control *); 
    int (*set_page_dirty)(struct page *page); 
    int (*readpages)(struct file *filp, struct address_space *mapping, 
                     struct list_head *pages, unsigned nr_pages);
    sector_t (*bmap)(struct address_space *, sector_t); 
}
```

writepage和writepages将地址空间的一页或多页写回到底层块设备。readpage和readpages从后备存储器将一页或多个连续的页读入页帧。sync_page对尚未回写到后备存储器的数据进行同步，该函数在块层的层次上运作，试图将仍然保存在缓冲区中的待决写操作写入到块层。而writepage在地址空间的层次上运作，只是将数据转发到块层，而不关注块层中的缓冲问题。bmap将地址空间内的逻辑块偏移量映射为物理块号。

page是与物理页相关的数据结构，内核用page描述当前时刻在相关的物理页中存放的东西，系统中的每个物理页都要分配一个page结构体。使用mapping和index字段即可在Page Cache中查找页。

```c
struct page  {
    unsigned long           flags;      // 标记页是否为脏，是否被锁定在内存等状态
    atomic_t                _count;     // 引用计数，为-1时表示内核并未引用，可回收
    atomic_t                _mapcount;
    unsigned long           private;
    struct address_space    *mapping;   // 与页关联的address_space对象
    pgoff_t                 index;      // 以页为单位的偏移量
    struct list_head        lru;
    void*                   virtual;    // 页的虚拟地址
};
```

分配页

```c
struct page * page_cache_alloc(struct address_space *x);
struct page * alloc_pages(gfp_t gfp_mask, unsigned int order);
```

gfp_mask标志分为行为修饰符、区修饰符和类型。行为修饰符指定了在分配时是否可以阻塞、使用高速缓存中快要淘汰出的页等，以便尽可能找到空闲的内存来满足分配请求。

page_cache_alloc根据address_space参数确定内存域，然后调用alloc_pages函数获得一个页帧。然后调用add_to_page_cache将新页添加到页缓存中。

查找页

```c
struct page * find_get_page(struct address_space *mapping, pgoff_t offset) { 
    struct page *page; 
    page = radix_tree_lookup(&mapping->page_tree, offset); 
    if (page) 
        page_cache_get(page); 
    return page; 
}
unsigned find_get_pages(struct address_space *mapping, pgoff_t start, 
                        unsigned int nr_pages, struct page **pages);
```

文件中的位置是按字节偏移量指定，首先将字节偏移量转换到页缓存偏移量，然后在基数树中搜索。

```c
struct page * find_or_create_page(struct address_space *mapping, 
                                  pgoff_t index, gfp_t gfp_mask);
```

find_or_create_page函数在页缓存中查找一页，如果没有则分配一个新页。

```c
int mpage_readpage(struct page *page, get_block_t get_block);
int mpage_readpages(struct address_space *mapping, struct list_head *pages, 
                    unsigned nr_pages, get_block_t get_block);
int mpage_writepage(struct page *page, get_block_t *get_block,
                    struct writeback_control *wbc);
int mpage_writepages(struct address_space *mapping, 
                     struct writeback_control *wbc, get_block_t get_block);
```

以mpage_readpages为例，mapping对应Page Cache，需要读入nr_pages个Page，循环将页添加到页缓存中，然后创建一个bio请求，从块读取所需的数据。

预读： 通常，普通文件是以相邻扇区成组存放在磁盘上，很少移动磁头就可以检索到文件。预读可以在实际请求前，读取文件的几个相邻的数据页，提高磁盘性能。但是预读不适用于随机文件。内核会自动根据读取Page Cache的命中情况，扩大或关闭预读窗口。

同步与交换：同步保证物理内存中保存的数据与后备存储器一致，而交换将导致从物理内存刷出数据，以释放空间，用于优先级更高的事项。在数据从物理内存清除之前，将与相关的后备存储器进行同步。

### 1.1 同步

Page Cache同步即回写磁盘，脏页只有经过同步之后才能被回收。

脏页写回磁盘的3种情况：

1. 用户进程调用sync或fsync手动清空缓冲
2. 空闲内存低于特定阈值
3. 脏页在内存中时间超过特定阈值或脏页比例超过特定阈值

相关的控制参数：

sysctl -a ｜ grep dirty

vm.dirty_background_bytes = 0 与ratio相关设置不会同时生效

vm.dirty_background_ratio = 10 当脏页占总内存的的百分比超过这个值时，后台线程开始刷新脏页。这个值如果设置得太小，可能不能很好地利用内存加速文件操作。如果设置得太大，则会周期性地出现一个写 IO 的峰值。

vm.dirty_bytes = 0

vm.dirty_expire_centisecs = 3000 脏数据多久会被刷新到磁盘上，3000即30s

vm.dirty_ratio = 20 当脏页占用的内存百分比超过此值时，内核会阻塞掉写操作，并开始刷新脏页

vm.dirty_writeback_centisecs = 500 多久唤醒一次刷新脏页的后台线程，500即5s

具体刷盘情况：

1. flusher线程在系统的空闲内存低于特定阈值时，将脏页刷回磁盘。内核调用flusher_threads()唤醒多个flusher线程，随后flusher线程调用bdi_writeback_all()将脏页刷回磁盘，直到满足以下条件（除非没有剩下的脏页可被写回了）：①已经有指定的最小数目的页被写出到磁盘 ②空闲内存已经回升，超过了阈值dirty_background_ratio
2. flusher线程被周期性唤醒，将那些在内存中驻留时间过长的脏页写出，确保内存中不会有长期存在的脏页。

2.6内核中使用pdflush。pdflush与flusher的区别：pdflush线程的数目是动态的，从2到8，采用拥塞回避机制，防止阻塞在同一个设备的拥塞队列上。flusher线程模型和具体块设备关联，所以每个给定线程从每个给定设备的脏页链表收集数据，并写回到对应磁盘，回写更趋于同步了。并且由于每个磁盘对应一个线程，所以线程不需要采用复杂的拥塞避免策略。该方法提高了I/O操作的公平性，而且降低了饥饿风险。

当达到vm.dirty_background_ratio时，后台线程将一定缓存的脏页**异步地刷入磁盘**，这一过程仍然可以进行写操作；如果多个应用进程写入Page Cache的速度快于后台线程向磁盘的回写速度，脏页比例达到vm.dirty_ratio时，系统会转为**同步地处理脏页的过程，阻塞应用进程，导致I/O延迟，直至比率刷到dirty_ratio之下，之后转为非阻塞的后台刷盘**。

根据工作负载，可以调整两者的值。同时调小两者的值，可以让数据更及时地刷到硬盘，减少在Page Cache缓存的数据；同时增大两者的值，会增大数据丢失和高I/O延迟的风险，适合多进程重复读写一个文件；减小vm.dirty_background_ratio，同时增大vm.dirty_ratio，适合处理突发的数据流，可以使更多的数据保存在Cache中使I/O处理更加平滑，但也会增大数据丢失的风险。

cat /proc/vmstat | egrep "dirty|writeback"

nr_dirty 需要写到磁盘的脏页数

### 1.2 交换与回收

当系统负载较低时，内存的大部分由Page Cache占用；当系统负载增加时，Page Cache会释放供进程使用。

通过页面回收，提供更多的主内存供其他程序使用。页面回收的三种方法：同步、交换和释放。页的刷出（flushing）、交换（swapping）、释放（releasing），不仅需要定期检查内存页的状态，还需要检查空闲内存的大小。未使用或很少使用的页将自动换出，但在换出前，其中包含的数据将与后备存储器同步，以防数据丢失。对动态产生的页，系统交换区充当后备存储器。对映射自文件的页来说，其交换区就是底层文件系统中与页对应的部分。如果内存发生严重的不足，必须强制刷出脏数据，以获得干净的页。

内存中的页分为：（1）不可回收页：保留页、内核页、内存锁定页等，不允许回收（2）可交换页：用户态地址空间的匿名页【如进程的用户态堆和栈中的页】等，回收时保存在交换区（3）可同步页：用户地址空间的映射页、存在磁盘文件数据且在Page Cache中的页等，需要进行同步。（4）可丢弃页

内存回收的2种情形：

- 内存紧缺回收：内核发现内存紧缺，例如无法获得新的缓冲区页、无法在给定的内存管理区分配一组连续页框。
- 周期回收：周期性地激活内核线程执行内存回收，kswapd内核线程检查某个内存管理区的空闲页框数是否低于pages_high值

内存回收算法：

- 双链策略：Page Cache的所有页被分成两组：活动链表与非活动链表。运行过程中，可以动态地将内存页加入两个链表，也可以在两个链表之间切换。页框回收算法会扫描非活动链表中的页面。

系统中的页数量可能非常大，上述的 LRU 链表也会非常长，每次扫描要回收的页时如果遍历整个链表，会花费很多时间，而且在内存比较充足的时候没必要进行大规模的回收。为了满足各种内存回收的场景，内核为内存回收过程定义了一个优先级 priority，priority 越趋向于 0 说明优先级越高，扫描的页数量就越多（注意扫描页数不等于回收页数，只有扫描到满足条件的页时，该页才会回收），默认的优先级是 12，如内存压力不是很大时就会以默认优先级进行内存回收。而当内存严重不足，就会以高优先级进行内存回收。除此之外在计算每个 LRU 链表的扫描页数时还要考虑 swappiness 参数，它会控制匿名内存 LRU 和页缓存 LRU 扫描页数的比例。

LRU链表中有两类页：用户态地址空间页、不属于任何用户态进程且在Page Cache中的页。页框回收程序倾向于压缩Page Cache，将用户态进程留在内存中。通过调整/proc/sys/vm/swappiness，确定移动所有的页还是只移动不属于用户态地址空间的页。为0时，不会回收用户态地址空间的页，为100时，每次调用都会回收用户态地址空间的页。

cat /proc/sys/vm/swappiness 控制页面回收时释放页缓存和释放匿名内存的比例，越大，使用交换区的积极性越高。默认值是 60，如果该值为 0，那么除非没有文件页缓存可以回收，否则不会进行页交换，而当该值为 100 时，页缓存和匿名内存会尽可能等比例的回收。

内存紧缺回收：

- 当分配VFS缓冲区失败时，唤醒脏页回写线程，触发脏页的写操作，使页框可释放
- 当伙伴系统分配失败时，会执行shrink_caches()和shrink_slab()进行页框回收，若失败，则会OOM删除一个进程

周期回收：

- 每个内存节点对应各自的kswapd内核线程，调用shrink_zone()和shrink_slab()从LRU链表中回收页，避免出现内存紧缺回收。当空闲页框数大于pages_high时，停止页框回收。

交换：为非映射页在磁盘上提供备份，例如进程匿名线性区（用户态堆栈），进程私有内存映射的脏页。交换是回收的高级特性，还可以用来扩展内存空间。但是进程对换出页的访问很慢，若要求性能，可以关闭交换功能。页交换只是一种紧急情况下的备用方案，它使得系统可以运行，但速度会降低很多。从内存中换出的页在交换区。如果页的后备存储器是一个块设备，则无需交换，而是与块设备同步；如果页的后备存储器是一个文件，但不能在内存中修改（例如，二进制可执行文件的数据），那么在当前不需要的情况下，可以直接丢弃该页。

**回收页面过程：**

对于干净文件页，使用unmap操作，从page cache中移除，即可进行回收；对于脏文件页，使用unmap操作，进行异步回写，然后从page cache中移除；对于脏非文件页，当扫描到此页，加入到swap cache，执行unmap操作，进行异步回写，然后将该页从swap cache中清除。