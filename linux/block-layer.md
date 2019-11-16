---
description: 这篇文章分析 Linux 块设备层。详细解释了 Linux 是如何处理块设备的。
---

# Linux 块设备

块设备是硬件设备，区别在于随机\(即，不一定是顺序\)访问固定大小的数据块。固定大小的数据块称为块。最常见的块设备是硬盘，但也存在许多其他块设备，如软盘驱动器、蓝光阅读器和闪存。请注意，这些都是安装文件系统的设备，文件系统是块设备的通用语言。

另一种基本类型的设备是字符设备。字符设备或字符设备作为连续数据流一个字节接一个字节地被访问。字符设备的例子有串口和键盘。如果硬件设备作为数据流被访问，则它被实现为字符设备。另一方面，如果设备是随机访问的\(非顺序\)，它就是块设备。

区别在于设备是否随机访问数据——换句话说，设备是否可以从一个位置寻找另一个位置。以键盘为例。作为驱动程序，键盘提供数据流。如果您键入wolf，键盘驱动程序会返回一个流，其中四个字母的顺序完全相同。无序阅读信件，或者阅读除了流中下一封之外的任何信件，都没有什么意义。键盘驱动器因此是充电设备；该设备提供用户在键盘上键入的字符流。从键盘上读取首先返回一个流，w，o，l，最后f。当没有按键等待时，该流为空。相反，硬盘驱动器则完全不同。硬盘驱动器可能要求读取一个任意块的内容，然后读取不同块的内容；这些块不必是连续的。硬盘的数据是随机存取的，而不是以流的形式；因此，硬盘是块设备。

管理内核中的块设备比管理角色设备需要更多的关注、准备和工作。字符设备只有一个位置——当前位置——而块设备必须能够在介质上的任何位置之间来回导航。 事实上，内核并不需要提供一个专门用于管理角色设备的完整子系统，但是块设备正好接收到这个。这种子系统是必要的，部分原因是块设备的复杂性。 然而，获得如此广泛支持的一个重要原因是块设备是相当性能敏感的；从硬盘中取出最后一滴比从键盘中挤出额外百分之一的速度要重要得多。 此外，正如您将看到的，块设备的复杂性为这种优化提供了很大的空间。本章的主题是内核如何管理块设备及其请求。内核的这一部分被称为块 IO 层。有趣的是，修改块 IO 层是2.5开发内核的主要目标。本章涵盖了2.6内核中全新的块 IO 层。

## 块设备的解剖

块设备上最小的可寻址单元是扇区。扇区有两种不同的幂，但512字节是最常见的大小。扇区大小是设备的物理属性，扇区是所有块设备的基本单位——设备不能在小于扇区的单位上寻址或操作，尽管许多块设备可以同时在多个扇区上操作。大多数块设备有512字节的扇区，尽管其他大小也很常见。例如，许多光盘有2千字节的扇区。

软件有不同的目标，因此强加了它自己最小的逻辑可寻址单元，即块block。块是文件系统的抽象——文件系统只能以块的倍数访问。虽然物理设备可以在扇区级寻址，但是内核执行所有以块为单位的磁盘操作。因为设备的最小可寻址单元是扇区，所以块大小不能小于扇区，并且必须是扇区的倍数。

块设备在硬件上一般最小单位是扇区，在操作系统层面最小单位是块。扇区的大小 &lt; 块的大小 &lt; 页的大小。

此外，内核\(与硬件和扇区一样\)需要该块是二的幂。内核还要求块不大于页面大小\(参见第12章“内存管理”和第19章"可移植性"\)。因此，块大小是扇区大小的2的幂倍并且要小于页大小。常见的块大小为 512 B, 1KB, 4KB。

有些令人困惑的是，有些人用不同的名字来指代扇区和区块。 扇区是设备最小的可寻址单元，有时被称为“硬扇区”或“设备块”同时，块，文件系统最小的可寻址单元，有时被称为“文件系统块”或“ I/O 块”本章继续称这两个概念为扇区和块，但是你应该记住其他术语。图14.1是扇区和缓冲器之间的关系图。 其他术语也很常见，至少在硬盘方面是如此——比如簇、圆柱体和磁头。这些概念只针对特定的块设备，并且在很大程度上对用户空间软件是不可见的。扇区对于内核来说之所以重要是因为在所有设备IO层面，所有的操作必须以扇区为单位。 所以你的块概念是构建在扇区概念的基础上的。

当一个块存储在内存中时——比如说，在读或写挂起之后——它被存储在缓冲区中。每个缓冲区只与一个块相关联。缓冲区充当表示内存中磁盘块的对象。回想一下，一个块由一个或多个扇区组成，但大小不超过一页。因此，单个页面可以在内存中保存一个或多个块。因为内核需要一些相关的控制信息来伴随数据\(例如来自哪个块设备和缓冲区是哪个特定块\)，所以每个缓冲区都与一个描述符相关联。描述符被称为缓冲头，属于`struct buffer_head`类型。buffer\_head结构保存内核操作缓冲区所需的所有信息，并在`<linux/buffer_head.h>`.中定义。 看看这个结构，用注释描述每个字段:

```cpp
struct buffer_head{
    unsigned long b_state; // 缓冲区状态标志
    struct buffer_head *b_this_page; // 页面缓冲区列表
    struct page *b_page; // 关联页面
    sector_t b_blocknr; // 起始块数
    size_t b_size; // 映射的大小
    char *b_data; // 指向页面内数据的指针
    struct block_device *b_bdev;  // 关联的块设备
    bh_end_io_t *b_end_io; //  IO 完成
    void *b_private; // 保留给 b_end_io
    struct list_head b_assoc_buffers; // 关联的映射
    struct address_space *b_assoc_map; // 关联地址空间
    atomic_t b_count; // 使用计数
};
```

`b_state`字段指定该特定缓冲区的状态。

它可以是表中的一个或多个标志。合法标志存储在`bh_state_bits`枚举中，该枚举在`<linux/buffer_head.h>`中定义。

表`bh_state`标志

| 状态 | 意义 |
| :--- | :--- |
| BH\_Uptodate | 缓冲区包含有效数据。 |
| BH\_Dirty | 缓冲区脏了。\(缓冲区的内容比磁盘上块的内容更新，因此缓冲区最终必须写回磁盘。\) |
| BH\_Lock | 缓冲区正在进行磁盘 I/O ，并被锁定以防止并发访问。 |
| BH\_Req | IO 请求中包含缓冲区。 |
| BH\_Mapped | 缓冲区是映射到磁盘块的有效缓冲区。 |
| BH\_New | 缓冲区是通过get\_block\(\)新映射的，尚未被访问。 |
| BH\_Async\_Read | 缓冲区正在通过`end_buffer_async_read()`异步读取 I/O |
| BH\_Async\_Write | 缓冲区正在通过`end_buffer_async_write()`异步写 I/O |
| BH\_Delay | 缓冲区还没有相关联的磁盘块\(延迟分配\)。 |
| BH\_Boundry | 缓冲区形成连续块的边界——下一个块是不连续的。 |
| BH\_Write\_EIO | 缓冲区在写入时发生 I/O 错误。 |
| BH\_Ordered | 有序写入。 |
| BH\_Eoptnotsupp | 缓冲区出现“不支持”错误。 |
| BH\_Unwritten | 磁盘上已分配了缓冲区空间，但实际数据尚未写出。 |
| BH\_Quiet | 抑制此缓冲区的错误。 |

`bh_state_bits`枚举还包含`BH_PrivateStart`标志作为列表中的最后一个值。
这不是有效的状态标志，而是对应于其他代码可以使用的第一个可用位。
块 I/O 层本身并不使用等于或大于`BH_PrivateStart`的所有位值，因此希望在`b_state`字段中存储信息的单个驱动程序可以安全地使用这些位。驱动程序可以根据该标志来确定其内部标志的位值，并确保它们不会侵犯块 I/O 层使用的官方位。
`b_count`字段是缓冲区的使用计数。该值由两个内联函数递增和递减，这两个函数都在`<linux/buffer_head.h>`中定义:

```cpp
static inline void get_bh(struct buffer_head *bh) {
    atomic_inc(&bh->b_count);
}

static inline void put_bh(struct buffer_head *bh) {
    atomic_dec(&bh->b_count);
}
```

在操作缓冲区头之前，您必须通过`get_bh()`增加它的引用计数，以确保缓冲区头不会从您下面解除分配。缓冲头用完后，通过`put_bh()`减少参考计数。

磁盘上给定缓冲区对应的物理块是`b_bdev`描述的块设备上的第`b_blocknr`个逻辑块。

内存中给定缓冲区对应的物理页面是`b_page`指向的页面。更具体地说，`b_data`是直接指向块(存在于`b_page`中的某个地方)的指针，其长度为`b_size`字节。因此，该块位于存储器中，从地址b_data开始，到地址`(b_data + b_size)`结束。

缓冲头的目的是描述磁盘块和物理内存缓冲区(特定页面上的字节序列)之间的映射。充当缓冲区到块映射的描述符是数据结构在内核中的唯一角色。

在2.6内核之前，缓冲头是一个更重要的数据结构:它是内核中的 IO 单元。缓冲头不仅描述了磁盘块到物理页面的映射，还充当了所有块 IO 的容器。首先，缓冲头是一个大而笨重的数据结构(现在它已经缩小了一点)，而且根据缓冲头来操作数据既不干净也不简单。相反，内核更喜欢以页面的形式工作，页面很简单，能够提高性能。描述每个单独缓冲区(可能小于一页)的大缓冲区头是低效的。因此，在2.6 内核中，已经做了很多工作让内核直接处理页面和地址空间，而不是缓冲区。第16章“页面缓存和页面写回”讨论了其中的一些工作，其中讨论了地址空间结构和`pdflush`守护程序。

缓冲头的第二个问题是它们只描述了一个缓冲区。当用作所有 I/O 操作的容器时，缓冲头迫使内核将潜在的大块 I/O 操作(比如写操作)分解成多个缓冲头结构。

这导致不必要的开销和空间消耗。因此，2.5开发内核的主要目标是为块 IO 操作引入一个新的、灵活的、轻量级的容器。结果是BIO结构，这将在下一节讨论。

## BIO 结构

内核中块 I/O 的基本容器是在`<linux/bio.h>`中定义的bio结构。该结构将正在运行(活动)的块 I/O 操作表示为段列表。段是内存中连续的缓冲区块。因此，单个缓冲区不需要在内存中是连续的。通过允许以块的形式描述缓冲区，bio结构为内核提供了从存储器的多个位置对甚至单个缓冲区进行逐块 I/O 操作的能力。像这样的矢量 IO 称为分散-聚集 IO 

以下是在`<linux/bio.h>`中定义的结构bio，为每个字段添加了注释:

```cpp
struct bio { 
    sector_t bi_sector;
    struct bio *bi_next;
    struct block_device *bi_bdev;
    unsigned long bi_flags;
    unsigned long bi_rw;
    unsigned short bi_vcnt;
    unsigned short bi_idx;
    unsigned short bi_phys_segments;
    unsigned int bi_size;
    unsigned int bi_seg_front_size;
    unsigned int bi_seg_back_size;
    unsigned int bi_max_vecs; 
    unsigned int bi_comp_cpu; 
    atomic_t bi_cnt; 
    struct bio_vec *bi_io_vec; 
    bio_end_io_t *bi_end_io; 
    void *bi_private; 
    bio_destructor_t *bi_destructor; 
    struct bio_vec bi_inline_vecs[0];
};
```

bio结构的主要目的是表示飞行中的块 IO 操作。为此，该结构中的大多数字段都与内务管理相关。最重要的字段是`bi_io_vec`、`bi_vcnt`和`bi_idx`。图14.2显示了bio结构与其朋友之间的关系。

## I/O 向量

bi_io_vec场指向一组bio_vec结构。在这个特定的块 IO 操作中，这些结构被用作单个段的列表。每个bio_vec都被视为`<page，offset，len>`形式的向量，它描述了一个特定的段:它所在的物理页、块在页中作为偏移量的位置以及从给定偏移量开始的块的长度。这些向量的完整数组描述了整个缓冲区。生物矢量结构在 `<linux/bio.h>` 中定义:

```cpp
struct bio_vec {
    /* pointer to the physical page on which this buffer resides */ 
    struct page *bv_page;
    /* the length in bytes of this buffer */ 
    unsigned int bv_len;
    unsigned int bv_offset; /* the byte offset within the page where the buffer resides */ 
};
```

在每个给定的块 I/O 操作中，在生物矢量阵列中有从生物矢量开始的双矢量。当执行块 I/O 操作时，bi_idx字段用于指向数组中的当前索引。

总之，每个块 IO 请求都由一个 bio 结构表示。每个请求由一个或多个存储在bio_vec结构阵列中的块组成。这些结构充当向量，描述每个段在内存物理页面中的位置。 I/O 操作中的第一段由b_io_vec指向。对于列表中的`bi_vcnt`段总数，每个附加段紧跟在第一个之后。当块 I/O 层提交请求中的段时，`bi_idx`字段被更新以指向当前段。

`bi_idx`字段用于指向列表中的当前bio_vec，这有助于块 I/O 层跟踪部分完成的块 I/O 操作。然而，更重要的用途是允许 bio 结构的分裂。利用此功能，实现廉价磁盘冗余阵列(磁盘阵列，一种硬盘设置，可使单个卷为了性能和可靠性而跨越多个磁盘)的驱动程序可以采用单个 bio 结构，最初是为单个设备设计的，并将其在磁盘阵列中的多个硬盘驱动器中进行分割。所有的磁盘阵列驱动程序需要做的就是复制io结构，并更新`bi_idx`字段，以指向单个驱动器应该开始其操作的位置。

 bio 结构在`bi_cnt`字段中保持使用计数。当该字段达到零时，结构被破坏，后备存储器被释放。以下两个功能为您管理使用计数器。

```cpp
void bio_get(struct bio *bio);
void bio_put(struct bio *bio);
```

前者增加使用计数，而后者减少使用计数(如果计数为零，则破坏 bio 结构)。在操纵飞行中的 bio 结构之前，一定要增加它的使用次数，以确保它不会完成并从你下面释放出来。完成后，依次减少使用次数。

最后，`bi_private`字段是结构所有者(即创建者)的私有字段。通常，只有分配了 bio 结构，才能读写该字段。

## Linus 电梯

现在让我们来看看一些现实生活中的 IO 调度器。第一个 IO 调度器叫做Linus电梯。\(是的，Linus有一部以他命名的电梯！\)它是2.4中的默认 I/O 调度程序。在2.6版本中，它被下面的 I/O 调度器所取代，我们将一直关注它，因为这台电梯比后面的那些更简单，同时执行许多相同的功能，它是一个很好的介绍。

Linus电梯执行合并和排序。 当一个请求被添加到队列中时，首先对照每隔一个挂起的请求进行检查，看它是否是合并的可能候选。 Linus电梯 IO 调度器执行前后合并。执行的合并类型取决于现有相邻请求的位置。 如果新请求立即继续现有请求，它将被预先合并。 相反，如果新请求紧接在现有请求之前，它将被重新合并。 由于文件的布局方式\(通常通过增加扇区号\)和在典型工作负载下执行的 I/O 操作\(数据通常从头到尾读取，而不是反向读取\)，与反向合并相比，前向合并很少。 然而，Linus电梯检查并执行两种类型的合并。

如果合并尝试失败，则在队列中寻找一个可能的插入点\(队列中新请求适合现有请求的扇区位置\)。如果找到一个，新的请求将被插入其中。如果没有找到合适的位置，请求将被添加到队列的尾部。 此外，如果在队列中发现的现有请求早于预定义的阈值，新请求将被添加到队列的尾部，即使它可以在其他地方进行插入排序。 这防止了对磁盘上邻近位置的许多请求无限期地减少对磁盘上其他位置的请求。 不幸的是，这种“年龄”检查没有效率。它不提供任何在给定时间范围内服务请求的实际尝试；它只是在适当的延迟后停止插入排序请求。这提高了延迟，但仍然会导致请求不足，这是2.4 I/O 调度程序的一个重要的必须解决的问题。

总之，当一个请求被添加到队列中时，四个操作是可能的。按照顺序，它们是

1. 如果对相邻磁盘扇区的请求在队列中，则现有请求和新请求合并成一个请求。
2. 如果队列中的一个请求足够旧，新请求将被插入队列尾部，以防止另一个较旧的请求饥饿。
3. 如果队列中有一个合适的扇区位置，则在那里插入新的请求。这将按照磁盘上的物理位置对队列进行排序。
4. 最后，如果不存在这样合适的插入点，请求将被插入队列的尾部。
