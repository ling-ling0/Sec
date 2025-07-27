区别于栈上内容, 堆部分与堆结构以及相关源代码关系密切, 所以这里首先对堆构造进行描述

此处描述的是linux系统所使用的glibc

#### chunk
chunk是glibc对内存空间管理的基本单位, `malloc`和`free`获取和释放的空间都以chunk的形式进行.

空闲chunk的基本结构:
```
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk, if unallocated (P clear)  |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
`head:' |             Size of chunk, in bytes                     |A|0|P|
  mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Forward pointer to next chunk in list             |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Back pointer to previous chunk in list            |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Unused space (may be 0 bytes long)                .
        .                                                               .
 next   .                                                               |
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
`foot:' |             Size of chunk, in bytes                           |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of next chunk, in bytes                |A|0|0|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
其在设计上选择了boundary tag形式, 两个相邻的chunk之间存在重叠部分. 使用`malloc`函数获取的返回地址是mem对应的地址.

对于空闲的chunk, 其会被修改之后添加到bins中, 等待下一次的申请内存. 在空闲状态的chunk, 其中fd和bk被设定其在bin链中的相邻chunk的地址.

prev_size记录的是物理地址相邻的前一个chunk的大小(如果前一个chunk是空闲的). 如果前一个chunk在使用中, 这部分被用作存储数据.

size必须是对齐的, 在64位系统中是8, 即大小必须为8的整数倍, 不足会向上补齐.

在使用中的chunk(没有被free掉)形式如下

```
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk, if unallocated (P clear)  |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of chunk, in bytes                     |A|M|P|
  mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             User data starts here...                          .
        .                                                               .
        .             (malloc_usable_size() bytes)                      .
next    .                                                               |
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             (size of chunk, but used for application data)    |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of next chunk, in bytes                |A|0|1|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

```

![alt text](images/useingchunk.png) 

其中每一个chunk会包含前一个chunk的大小和这个chunk的大小. 在使用中的时候, 其中的大部分空间都会被用来作为数据存储空间, 在空闲时的指针内容会被覆盖掉.

当一个 chunk 处于已分配状态时, 它的物理相邻的下一个 chunk 的 prev_size 字段必然是无效的, 故而这个字段就可以被当前这个 chunk 使用. 这是 ptmalloc 中 chunk 间的复用。

#### bins
被释放的chunk不会立即交回给系统, 而是由ptmalloc统一管理heap和mmap映射区域中的空闲chunk. 当再次请求分配内存时, ptmalloc会先在空闲的chunk中寻找一个合适的给用户.

ptmalloc中空闲的chunk使用bin进行管理. bin主要分为4种, fast bins, small bins, large bins, unsorted bins. 在每一种bins中有更详细地划分, 每一个划分使用一个链表将chunk串联起来. 

对于fast bins, ptmalloc中会将一些常用大小的内存块在释放时直接放入fast bins, 这部分使用单向链表进行存储, 并采用FILO策略, 使得获取内存块的操作更为迅速. 在fast bin中, 保存几种属于small bin范围中大小的chunk, 如32B, 48B等.
其使用一个数组保存, 每个元素存储一条链表.

对于small, large, unsorted bins, 其链表同样被存储在一个数组中:

```#define NBINS 128
/* Normal bins packed as described above */
mchunkptr bins[ NBINS * 2 - 2 ];
```

其中每个元素是一条链表的起始地址. 第一个是unsorted bin, 然后是62个small bins, 最后是63个large bins.

其中unsorted bin用来存储未排序的chunk. 当一个大的chunk分割成小chunk时, 剩下的部分就会先加入unsorted bin中, fast bin中的chunk被清空之后也会加入unsorted bin. 这部分chunk在合适的时候会重新加入small or large bins中.

unsorted bin采用FIFO策略, 并使用双向链表保存. 任何大小的chunk都可以放入unsorted bin中.

small bin和large bin相似, 都是使用双向链表存储固定大小的chunk. 其中按照固定的大小分为62+63个bin

#### top chunk
在ptmalloc中存在top chunk. 其是在地址上最高的chunk. 其不属于任何bin. 最开始其占据heap的所有空间, 当第一次执行malloc时, 这个chunk会分出一个需要的大小的chunk返回, 剩下的部分仍作为top chunk. 

top chunk主要就是当目前所有bin中的chunk都无法满足需求的时候, 会从top chunk中再分出一个chunk使用. 如果整个空间比作一桶水, 这部分就是一直没有被使用的部分, 外面的都不能用的时候才会从这里取出一部分.

如果top chunk也满足不了用户的需求, 此时就会将heap的空间进行扩展后再进行分配. 在 main arena 中通过 sbrk 扩展 heap, 而在 thread arena 中通过 mmap 分配新的 heap.

#### arena
arena是管理堆的结构, 也就是管理上面所有内容的部分. 每个线程只有一个arena, 主线程的arena是main_arena, 子线程的arena是thread_arena. 系统限制arena的最大数量.

每个线程首次调用malloc时会被glibc分配一个arena. 如果系统可用的arena数量为0, 该线程会被阻塞, 直到有可以使用的arena.

main_arena只能有一个堆, 而thread_arena可以有多个堆.

对于thread_arena, 其中包含heap_info(每个heap的详细信息), malloc_state(也叫arena header, 包含bins, top chunk等信息)和malloc_chunk(就是上面的chunk).

而在main_arena中只会有一块内存, 当需要更多的时候只会从这个空间继续扩展(使用brk), 所以其没有heap_info这部分.