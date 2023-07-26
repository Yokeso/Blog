---
title: Linux内核设计与实现ch14  块IO层
date: 2023-07-20 15:11:22
tags: [Linux内核设计与实现, 块IO层, Linux内核]
---

# Linux内核设计与实现第十四章--块I/O层

块设备是系统中能够随机访问固定大小数据片的硬件设备，所以这些固定大小的数据片就称为块。最常见的块设备是硬盘，其余闪存，光驱等也均为块设备。在内核中，这些块设备都是以安装文件系统的方式使用的。

与块设备相对应的设备类型是字符设备。字符设备会按照字节流的方式被有序访问（例如串口或者键盘这样的设备）。也就是说字符设备和块设备之间的最主要区别就是——数据是否可以被随机访问。

对于内核来讲，对于块设备的管理要比对于字符设备的管理细致的多。因为块设备中要考虑的问题比字符设备更多。字符设备只需要控制当前位置的数据，而块设备上的位置需要在介质的区间中不停移动。更重要的是，块设备普遍来说对执行性能的要求很高。对硬盘的每多一份利用都会对整个系统带来性能上的提升。所以内核给块设备提供了专门提供服务的子系统。而这一章讲述的就是内核如何对块设备和块设备的请求进行管理。

<!--more-->

### 14.1 刨析一个块设备

块设备中最小的可寻址单元是扇区，其大小一般为2的整数倍，最常见的是512字节。扇区的大小是设备的物理属性，扇区是所有块设备的基本单元——块设备无法对比它还小的单元进行寻址和操作，但很多块设备可以一次对多个扇区进行操作。

由于内核执行的所有磁盘操作都是按照块进行访问的，所以块概念不能比扇区还要小，通常是倍数于扇区大小。并且块大小要是扇区大小的二的整数倍。所以通常来说块定义为512字节，1KB或者4KBDD

### 14.2 缓冲区和缓冲头

当一个块被调入内存时，它要存储在一个缓冲区中。每个缓冲区与一个块对应，相当于是磁盘块在内存中的表示。块包含了一个或多个扇区，但是不能超过页大小。由于内核在处理数据时需要一些相关的控制信息（块属于哪个块设备、块属于哪个缓冲区等），所以每个缓冲区都要有一个对应的描述符`buffer_head`来表示。该缓冲区在`<linux/buffer_head.h>`中定义，描述了磁盘块到物理内存缓冲区之间的映射关系。

> 在2.6内核之前，缓冲头是一个更重要的多的数据结构。除了描述了磁盘块到物理内存的映射之外，还作为所有块I/O操作的容器存在。但是这样就使得这个结构体的大小很大，而且操作起来不清晰也不方便。并且会导致效率低下。所以从2.6内核开始，为块IO引入了一种新型，灵活且轻量的容器。这个我们再后面一节再介绍

```c
struct buffer_head {
        unsigned long b_state;          /* buffer state bitmap (see above) */
        struct buffer_head *b_this_page;/* circular list of page's buffers */
        struct page *b_page;            /* the page this bh is mapped to */

        sector_t b_blocknr;             /* start block number */
        size_t b_size;                  /* size of mapping */
        char *b_data;                   /* pointer to data within the page */

        struct block_device *b_bdev;
        bh_end_io_t *b_end_io;          /* I/O completion */
        void *b_private;                /* reserved for b_end_io */
        struct list_head b_assoc_buffers; /* associated with another mapping */
        struct address_space *b_assoc_map;      /* mapping this buffer is
                                                   associated with */
        atomic_t b_count;               /* users using this buffer_head */
};
```

其中`b_state`是多种标志的组合，合法的标志存放在一个名为`bh_state_bits`的枚举中，该枚举同样在该文件中定义

```c
enum bh_state_bits {
        BH_Uptodate,    /* Contains valid data */
        BH_Dirty,       /* Is dirty */
        BH_Lock,        /* Is locked */
        BH_Req,         /* Has been submitted for I/O */
        BH_Uptodate_Lock,/* Used by the first bh in a page, to serialise
                          * IO completion of other buffers in the page
                          */

        BH_Mapped,      /* Has a disk mapping */
        BH_New,         /* Disk mapping was newly created by get_block */
        BH_Async_Read,  /* Is under end_buffer_async_read I/O */
        BH_Async_Write, /* Is under end_buffer_async_write I/O */
        BH_Delay,       /* Buffer is not yet allocated on disk */
        BH_Boundary,    /* Block is followed by a discontiguity */
        BH_Write_EIO,   /* I/O error on write */
        BH_Unwritten,   /* Buffer is allocated on disk but not written */
        BH_Quiet,       /* Buffer Error Prinks to be quiet */
        BH_Meta,        /* Buffer contains metadata */
        BH_Prio,        /* Buffer should be submitted with REQ_PRIO */
        BH_Defer_Completion, /* Defer AIO completion to workqueue */

        BH_PrivateStart,/* not a state bit, but the first bit available
                         * for private allocation by other entities
                         */
};
```

需要注意的是`BH_PrivateStart`，该标志不是状态标志。而是用来指明可被其他代码使用的起始位。块I/O层不会使用`BH_PrivateStart`或者更高位，那么当驱动程序希望通过`b_state`域存储信息时就可以安全使用这些位进行状态自定义。只要保证自定义的标志状态不和块IO层的专用位起冲突就可以了。

`b_count`域代表缓冲区的使用用户计数，通过内联函数进行增减。

```c
static inline void get_bh(struct buffer_head *bh)
{
        atomic_inc(&bh->b_count);
}

static inline void put_bh(struct buffer_head *bh)
{
        smp_mb__before_atomic();
        atomic_dec(&bh->b_count);
}
```

> 要注意的是，在操作缓冲区头之前要先用`get_bh()`增加缓冲区头的引用计数，确保该缓冲区头不会再被分配出去；当完成对缓冲区头的操作之后，还需要用`put_bh()`减少引用计数。

`b_blocknr`用于索引与缓冲区相对应的物理块，其值是`b_bdev`域指明的块设备中的逻辑块号。缓冲区对应的物理页则用`b_page`域表示，而`b_data`则直接指向位于`b_page`域指明的页面上的对应的块。块的大小则由`b_size`来表示，起始位置在`b_data`处，结束位置在`b_data + b_size`处。

### 14.3 bio结构体

内核中的块I/O操作的基本容器由bio结构体来表示，该结构体定义在文件`<linux/blk_types.h>`中（内核5.4）。代表了正在活动的以`segment`链表形式组织的块I/O操作。这里的一个`segment`是一小块连续的缓冲区。这样以来就可以通过`segment`来描述缓冲区，即使缓冲区分散在多个位置上，bio结构体也能对内核保证I/O操作的执行。从而保证了单个缓冲区不一定要连续。这样的I/O我们起了一个名称叫做**聚散I/O**。

由于本节中的bio结构体与2.6内核中的bio结构体变化较大，所以这节按照5.4内核来进行分析。下面给出bio结构体的各个域的描述：

```c
struct bio {
        struct bio              *bi_next;       /* request queue link */
        struct gendisk          *bi_disk;
        unsigned int            bi_opf;         /* bottom bits req flags,
                                                 * top bits REQ_OP. Use
                                                 * accessors.
                                                 */
        unsigned short          bi_flags;       /* status, etc and bvec pool number */
        unsigned short          bi_ioprio;
        unsigned short          bi_write_hint;
        blk_status_t            bi_status;
        u8                      bi_partno;

        struct bvec_iter        bi_iter;

        atomic_t                __bi_remaining;
        bio_end_io_t            *bi_end_io;

        void                    *bi_private;
#ifdef CONFIG_BLK_CGROUP
        /*
         * Represents the association of the css and request_queue for the bio.
         * If a bio goes direct to device, it will not have a blkg as it will
         * not have a request_queue associated with it.  The reference is put
         * on release of the bio.
         */
        struct blkcg_gq         *bi_blkg;
        struct bio_issue        bi_issue;
#ifdef CONFIG_BLK_CGROUP_IOCOST
        u64                     bi_iocost_cost;
#endif
#endif
        union {
#if defined(CONFIG_BLK_DEV_INTEGRITY)
                struct bio_integrity_payload *bi_integrity; /* data integrity */
#endif
        };

        unsigned short          bi_vcnt;        /* how many bio_vec's */

        /*
         * Everything starting with bi_max_vecs will be preserved by bio_reset()
         */
    
        unsigned short          bi_max_vecs;    /* max bvl_vecs we can hold */

        atomic_t                __bi_cnt;       /* pin count */

        struct bio_vec          *bi_io_vec;     /* the actual vec list */

        struct bio_set          *bi_pool;

        /*
         * We can inline a number of vecs at the end of the bio, to avoid
         * double allocations for a small number of bio_vecs. This member
         * MUST obviously be kept at the very end of the bio.
         */
        struct bio_vec          bi_inline_vecs[0];
};
```

bio结构体的目的是代表正在现场执行的io操作。所以结构体中的主要域都是管理相关信息的。这里面最重要的几个域是`bi_io_vec`、`bi_vcnt`以及`bi_idx`(这个位于`struct bvec_iter bi_iter`中)。`bi_io_vec`是现场执行的页的结构体链表，其总数为`bio_vcnt`。而`bi_idx`则指向了目前在用的页。

`bi_io_vec`域指向一个结构体链表，该链表的每一个节点都是`bio_vec`结构体。该结构体的成员组成了一个`<page, offset, len>`的向量。在每个给定的块I/O操作中，和前面刚描述完的一样，`bio_vcnt`用来描述`bi_io_vec`所指向的`vio_vec`数组中的向量数目。而`bi_idx`指向数组的当前索引。

块I/O层可以通过跟踪`bi_idx`来了解I/O操作的完成进度。但该域更重要的作用是用于分割bio结构体，例如RAID可以将单独的bio结构体分割到RAID阵列上的各个硬盘中去。RAID驱动只需要拷贝这个bio结构体，然后将`bi_idx`域设置为每个独立硬盘操作时需要的位置就可以了。

`bi_cnt`域记录了bio结构体的使用计数，如果其值减为0，则bio结构体将会被撤销并进行内存释放。这个域通过以下两个函数来进行管理：前者增加使用计数，后者减少使用计数。

```c
void bio_get(struct bio *bio);
void bio_put(struct bio *bio);
```

还有一个需要注意的域是`bi_private`域。该域为创建者所属的私有域。只有创建了bio结构的拥有者可以对该域进行读写。

### 14.4 请求队列

块设备将他们挂起的块I/O请求保存在请求队列中，该队列由`reques_queue`来表示，定义在文件`<linux/blkdev.h>`中。包含了一个双向链表及其相关的控制信息。通过内核中的文件系统这样的高层代码添加到请求队列中。只要队列不为空，队列对应的块设备驱动就会从队列头获取请求，然后将其送入对应的块设备上去。请求队列表中的每一项都是一个单独的请求，由`request`结构体来表示。

`request`结构体定义在`<linux/blkdev.h>`中，由于一个请求可能要设计操作多个连续的盘块，所以每个请求可以由多个bio结构体组成。虽然磁盘上的块必须连续，但是在内存中这些块并不一定要连续——每个bio结构体都可以描述多个片段。

### 14.5 I/O调度程序

磁盘寻址是整个计算机中最慢的操作之一。每一次寻址都需要花费大量的时间。所以对于系统来说，系统性能提升的关键路径之一就是缩短寻址时间。

为了优化寻址操作，内核既不会简单的请求接收次序，也不会立即将其提交给磁盘，而是在提交前进行合并与排序的预操作。而在内核中用于提交I/O请求的子系统称为I/O 调度程序。

> I/O 调度程序通过将请求队列中挂起的请求进行合并与排序将磁盘I/O资源分配给系统中所有挂起的块I/O请求。这里的调度程序要注意与第四章中描述的进程调度区分开，进程调度是将处理器的资源分配给系统中正在进行的进程，而I/O调度程序则是对虚拟块设备的资源整合，给多个磁盘请求，以降低磁盘寻址时间，确保磁盘性能最优化。

#### 14.5.1 I/O调度程序的工作

I/O调度程序的工作是管理块设备的请求队列。它决定了队列中的请求排列顺序以及在什么时刻派发请求到块设备。这样做有利于减少磁盘寻址时间，提高全局吞吐量（之所以说提高全局吞吐量是因为一个I/O调度器可能因为系统整体性能考虑从而不公平对待所有请求）

前面提到过，I/O调度程序通过两种办法来减少磁盘寻址时间：合并与排序。让我们对这两个情况分别举一个例子：

+ 合并指的是将多个请求进行结合生成一个新的请求。假定两次请求中需要的块刚好是相邻扇区，那么这两个请求就可以合并为一个请求，从而将多次寻址开销转化为一次寻址开销。这种合并显然能减少系统的开销以及磁盘的寻址次数。

+ 排序则是对多个请求进行重排序操作。假定存在一个请求，他所请求的磁盘操作的扇区与当前请求比较接近，那么显然让他们进行相邻请求更加合理。所以说I/O调度程序做了这样一件事情：将整个请求队列按照扇区增长的方向有序排列。尽可能使得所有请求按照硬盘上扇区的排列顺序有序。这一操作的目的不仅仅为了缩短单独一次请求的寻址时间，更重要的是通过磁盘头直线方向的移动来缩短磁盘所需的寻道时间。这一算法类似于电梯的调度——电梯不能随意跳跃楼层，而是应该向一个方向移动到需求的最大后再向另外一个方向移动。处于这种相似性，I/O调度算法也被称为电梯调度。

#### 14.5.2 Linus电梯

Linus电梯再现在的内核中已经不复存在了（2.6版本开始被替换掉）。但是由于这个电梯的思想比其他的调度程序都简单不少，而且执行功能有很多相似，所以可以作为一个入门程序来进行学习。

Linus电梯能执行合并于排序的预处理。当有新请求加入队列时，它首先会检查其他每一个挂起的请求是否可以和新的请求进行合并。Linus电梯I/O调度程序可以执行向前和向后合并，合并描述了请求是向前还是向后的，并与现有的请求相连。