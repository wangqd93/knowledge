## redis

### 常见的数据类型

#### string: 动态字符串，是可以修改的字符串，三种存储分别为embstr、raw、int

+ 当字符串长度小于1M时，扩容都是加倍现有的空间，如果超过1M，扩容时一次只会多扩1M的空间。需要注意的是字符串最大长度为512M

+ 长度小于等于44位embstr，大于raw 
  + embstr
    + 16个字节的robj结构。
    + 3个字节的sdshdr8头。
    + 最多44个字节的sds字符数组。
    + 1个NULL结束符。

  + 16字节redisObject 占用，3字节SDS，1字节结尾\0, 64-16-3-1 = 44
  + embstr编码则通过调用一次内存分配函数来分配一块连续的空间
  + embstr 是不可变的，当使用append key value 指令的时候，embstr 会转为 raw


#### hash: 适合存储相关的一组key，失效时间只能是1个

当存储的数据量较少的时，hash 采用 ziplist 作为底层存储结构，此时要求符合以下两个条件：

- 哈希对象保存的所有键值对（键和值）的字符串长度总和小于 64 个字节。
- 哈希对象保存的键值对数量要小于 512 个。

hash 就会采用第二种方式来存储数据，也就是 dict（字典结构），该结构类似于 Java 的 HashMap，是一个无序的字典，并采用了数组和链表相结合的方式存储数据。在 Redis 中，dict 是基于哈希表算法实现的，因此其查找性能非常高效，其时间复杂度为 O(1)。

#### list:

列表（`list`）类型是用来存储多个 **有序** 的 **字符串**。在 `Redis` 中，可以对列表的 **两端** 进行 **插入**（`push`）和 **弹出**（`pop`）操作，还可以获取 **指定范围** 的 **元素列表**、获取 **指定索引下标** 的 **元素** 等。

ziplist（压缩列表）

当列表的元素个数 **小于** `list-max-ziplist-entries` 配置（默认 `512` 个），同时列表中 **每个元素** 的值都 **小于**  `list-max-ziplist-value` 配置时（默认 `64` 字节），`Redis` 会选用 `ziplist` 来作为列表的内部实现来减少内存的使用。

linkedlist（链表）

当列表类型无法满足 `ziplist` 的条件时， `Redis` 会使用 `linkedlist` 作为 列表 的 内部实现。

#### set:

集合（`set`）类型也是用来保存多个 **字符串元素**，但和 **列表类型** 不一样的是，集合中 **不允许有重复元素**，并且集合中的元素是 **无序的**，不能通过 **索引下标** 获取元素。

intset（整数集合）

当集合中的元素都是 整数且 元素个数小于 `set-max-intset-entries` 配置（默认 `512` 个）时，`Redis` 会选用 `intset` 来作为 集合 的内部实现，从而减少内存 的使用。

hashtable（哈希表）

当集合类型 **无法满足** `intset` 的条件时，`Redis` 会使用 `hashtable` 作为集合的 **内部实现**。

#### zset

实际上，Redis中sorted set的实现是这样的：

- 当数据较少时，sorted set是由一个ziplist来实现的。
- 当数据多的时候，sorted set是由一个dict + 一个skiplist来实现的。简单来讲，dict用来查询数据到分数的对应关系，而skiplist用来根据分数查询数据（可能是范围查找）。

### 底层数据支持

#### dict

http://zhangtielei.com/posts/blog-redis-dict.html

#### SDS( Simple dynamic string)

参考：http://zhangtielei.com/posts/blog-redis-sds.html

```c
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

len: 表示字符串的真正长度（不包含NULL结束符在内）。

alloc: 表示字符串的最大容量（不包含最后多余的那个字节）。

flags: 总是占用一个字节。其中的最低3个bit用来表示header的类型。header的类型共有5种，在sds.h中有常量定义。

```c
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```

![img](http://zhangtielei.com/assets/photos_redis/redis_sds_structure.png)

buf[]:数组



#### robject

用于存储类型，encode

```c
/* Object types */
#define OBJ_STRING 0
#define OBJ_LIST 1
#define OBJ_SET 2
#define OBJ_ZSET 3
#define OBJ_HASH 4

/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */

#define LRU_BITS 24
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount;
    void *ptr;
} robj;
```

一个robj包含如下5个字段：

- type: 对象的数据类型。占4个bit。可能的取值有5种：OBJ_STRING, OBJ_LIST, OBJ_SET, OBJ_ZSET, OBJ_HASH，分别对应Redis对外暴露的5种数据结构（即我们在第一篇文章中提到的第一个层面的5种数据结构）。
- encoding: 对象的内部表示方式（也可以称为编码）。占4个bit。可能的取值有10种，即前面代码中的10个OBJ_ENCODING_XXX常量。
- lru: 做LRU替换算法用，占24个bit。这个不是我们这里讨论的重点，暂时忽略。
- refcount: 引用计数。它允许robj对象在某些情况下被共享。
- ptr: 数据指针。指向真正的数据。比如，一个代表string的robj，它的ptr可能指向一个sds结构；一个代表list的robj，它的ptr可能指向一个quicklist。

encoding类型

- OBJ_ENCODING_RAW: 最原生的表示方式。其实只有string类型才会用这个encoding值（表示成sds）。
- OBJ_ENCODING_INT: 表示成数字。实际用long表示。
- OBJ_ENCODING_HT: 表示成dict。
- OBJ_ENCODING_ZIPMAP: 是个旧的表示方式，已不再用。在小于Redis 2.6的版本中才有。
- OBJ_ENCODING_LINKEDLIST: 也是个旧的表示方式，已不再用。
- OBJ_ENCODING_ZIPLIST: 表示成ziplist。
- OBJ_ENCODING_INTSET: 表示成intset。用于set数据结构。
- OBJ_ENCODING_SKIPLIST: 表示成skiplist。用于sorted set数据结构。
- OBJ_ENCODING_EMBSTR: 表示成一种特殊的嵌入式的sds。
- OBJ_ENCODING_QUICKLIST: 表示成quicklist。用于list数据结构。

#### ziplist结构

http://zhangtielei.com/posts/blog-redis-ziplist.html

ziplist是一个经过特殊编码的双向链表，它的设计目标就是为了提高存储效率。ziplist可以用于存储字符串或整数，其中整数是按真正的二进制表示进行编码的，而不是编码成字符串序列。它能以O(1)的时间复杂度在表的两端提供`push`和`pop`操作。

ziplist是将表中每一项存放在前后连续的地址空间内，一个ziplist整体占用一大块内存。它是一个表（list），但其实不是一个链表（linked list）

从宏观上看，ziplist的内存结构如下：

```
<zlbytes><zltail><zllen><entry>...<entry><zlend>
```

各个部分在内存上是前后相邻的，它们分别的含义如下：

- `<zlbytes>`: 32bit，表示ziplist占用的字节总数（也包括`<zlbytes>`本身占用的4个字节）。
- `<zltail>`: 32bit，表示ziplist表中最后一项（entry）在ziplist中的偏移字节数。`<zltail>`的存在，使得我们可以很方便地找到最后一项（不用遍历整个ziplist），从而可以在ziplist尾端快速地执行push或pop操作。
- `<zllen>`: 16bit， 表示ziplist中数据项（entry）的个数。zllen字段因为只有16bit，所以可以表达的最大值为2^16-1。这里需要特别注意的是，如果ziplist中数据项个数超过了16bit能表达的最大值，ziplist仍然可以来表示。那怎么表示呢？这里做了这样的规定：如果`<zllen>`小于等于2^16-2（也就是不等于2^16-1），那么`<zllen>`就表示ziplist中数据项的个数；否则，也就是`<zllen>`等于16bit全为1的情况，那么`<zllen>`就不表示数据项个数了，这时候要想知道ziplist中数据项总数，那么必须对ziplist从头到尾遍历各个数据项，才能计数出来。
- `<entry>`: 表示真正存放数据的数据项，长度不定。一个数据项（entry）也有它自己的内部结构，这个稍后再解释。
- `<zlend>`: ziplist最后1个字节，是一个结束标记，值固定等于255。

#### quicklist

http://zhangtielei.com/posts/blog-redis-quicklist.html

quicklist的每个节点都是一个ziplist。

quicklist的结构为什么这样设计呢？总结起来，大概又是一个空间和时间的折中：

- 双向链表便于在表的两端进行push和pop操作，但是它的内存开销比较大。首先，它在每个节点上除了要保存数据之外，还要额外保存两个指针；其次，双向链表的各个节点是单独的内存块，地址不连续，节点多了容易产生内存碎片。
- ziplist由于是一整块连续内存，所以存储效率很高。但是，它不利于修改操作，每次数据变动都会引发一次内存的realloc。特别是当ziplist长度很长的时候，一次realloc可能会导致大批量的数据拷贝，进一步降低性能。

于是，结合了双向链表和ziplist的优点，quicklist就应运而生了。

#### skiplist

跳表

### redis rehash

Redis的dict实现最显著的一个特点，就在于它的重哈希。它采用了一种称为增量式重哈希（incremental rehashing）的方法，在需要扩展内存时避免一次性对所有key进行重哈希，而是将重哈希操作分散到对于dict的各个增删改查的操作中去。这种方法能做到每次只对一小部分key进行重哈希，而每次重哈希之间不影响dict的操作。dict之所以这样设计，是为了避免重哈希期间单个请求的响应时间剧烈增加，这与前面提到的“快速响应时间”的设计原则是相符的。

### redis 快的原因

1.Redis 大部分操作是在内存上完成

2.Redis 采用多路复用，能保证在网络 IO 中可以并发处理大量的客户端请求，实现高吞吐率

### redis事件模型

redis服务器是一个事件驱动的程序，内部需要处理两类事件，

一类是文件事件**（file event）**，一类是时间事件**（time event）**，前者对应着处理各种io事件，后者对应着处理各种定时任务。

Redis中**文件事件包括**：客户端的连接、命令请求、数据回复、连接断开等，当上述事件发生时，会造成相应的描述符可读可写，再调用相应类型的文件事件处理器。

文件事件处理器有：

　　　连接应答处理器`networking.c/acceptTcpHandler`；

　　　命令请求处理器`networking.c/readQueryFromClinet`；

　　　命令回复处理器`networking.c/sendReplyToClient`；

时间事件（time event）：时间事件包含`定时事件`和`周期性事件`，Redis将其放入一个单向无序链表中，每当时间事件执行器运行时，就遍历链表，查找已经到达的时间事件，调用相应的处理器。

　　　　**定时事件**：让一段程序在指定的时间之后执行一次，比如让程序X在当前时间30ms之后执行一次；

　　　　**周期事件**：让一段程序每隔指定时间就执行一次，比如让程序Y每隔30ms就执行一次；



file event 底层其实是通过**select/epoll等模型**去执行驱动的，time event是通过**内部维持一个定时任务**列表来实现的。

### redis集群

#### 哨兵模式

#### 集群模式



### 其他缓存技术

spring 本地缓存



### 如何保证缓存一致性

先更新 再删除



### redis 项目中用的方式

项目中用的redission

主要用的方法：get、set、setNX、setEX、exists、incr、decr、tryLock、del

 