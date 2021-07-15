# Redis 基础系列（四）—— 哈希对象

在 Redis 中， 哈希类型是指键值本身又是一个键值对结构， 形如 **value={{field1， value1}， ...{fieldN， valueN}}**。Redis 中，每个哈希键可以存储键值对数量为

$$
2 ^ {23} −1
$$

个。

## 哈希对象编码

哈希对象内部编码有两种： `ziplist` 和 `hashtable` 。

- `ziplist` (压缩列表)：当哈希对象元素个数小于`hash-max-ziplist-entries`配置（默认为 512 个），同时所有值的长度小于 `hash-max-ziplist-value` 配置（默认为 64 字节）时，Redis 内部会使用 `ziplist`编码存储哈希对象。每当有新的键值对要加入到哈希对象时， 程序会先将保存了键的压缩列表节点推入到压缩列表表尾， 然后再将保存了值的压缩列表节点推入到压缩列表表尾。

  - 保存了同一键值对的两个节点总是紧挨在一起， 保存键的节点在前， 保存值的节点在后；
  - 先添加到哈希对象中的键值对会被放在压缩列表的表头方向， 而后来添加到哈希对象中的键值对会被放在压缩列表的表尾方向。

- `hashtable`（字典) ： 当哈希类型无法满足`ziplist`的条件时， Redis 会使用`hashtable`作为哈希的内部实现， 因为此时 ziplist 的读写效率会下降， 而`hashtable`的读写时间复杂度为 O(1).

## 字典

Redis 的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的每个健值。

```c
typedef struct dictEntry {

    void *key;

    union {

        void *val;

        uint64_t u64;

        int64_t s64;

        double d;

    } v;

    struct dictEntry *next;

} dictEntry;



typedef struct dictType {

    uint64_t (*hashFunction)(const void *key);

    void *(*keyDup)(void *privdata, const void *key);

    void *(*valDup)(void *privdata, const void *obj);

    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    void (*keyDestructor)(void *privdata, void *key);

    void (*valDestructor)(void *privdata, void *obj);

} dictType;



/* This is our hash table structure. Every dictionary has two of this as we

 * implement incremental rehashing, for the old to the new table. */

typedef struct dictht {

    dictEntry **table;

    unsigned long size;

    unsigned long sizemask;

    unsigned long used;

} dictht;



typedef struct dict {

    dictType *type;

    void *privdata;

    dictht ht[2];

    long rehashidx; /* rehashing not in progress if rehashidx == -1 */

    unsigned long iterators; /* number of iterators currently running */

} dict;
```

### 哈希算法

Redis 计算哈希值和索引值的方法如下：

- 使用字典设置的哈希函数，计算键 key 的哈希值
- hash = dict -> type -> hashFunction(key)
- 使用哈希表的 sizemask 属性和哈希值，计算出索引值(sizemask 为哈希表大小掩码，用于计算索引值，总是等于 size - 1)
- 根据情况不同，ht[x]可以是 ht[0]或者 ht[1]
- index = hash & dict -> ht[x].sizemask

### 解决键冲突

当有两个或两个以上数量的键被分配到了哈希表数组的同一个索引上面时，我们称这些键发生了冲突。

- 解决哈希冲突的四种方法
  - 开放地址法
    - 线性探测法：在冲突的值上加上一个单位的值，直至不冲突。
    - 再平方探测法：在冲突的值上加上 1 的平方个单位，如果冲突则减去 1 的平方个单位；接着 2 的平方、3 的平方，直至不冲突。
    - 伪随机法：在冲突的值上加上一个随机数，直至不冲突。
  - 链式地址法：对冲突的的值使用功能链表方式存储
    - 处理方式简单，不会产生堆积现象，平均查找长度较短。
    - 链表节点可以随意扩展，适合无法确定长度的情况。
    - 相较于开放地址法，链式地址法更节省空间。
  - 建立公共溢出区：建立公共溢出区存储所有冲突的值。
  - 再哈希法：对于冲突的值再次使用哈希算法，直至不发生冲突。

Redis 使用联地址法解决键冲突，每个哈希表节点都有一个 next 指针，多个哈希表节点可以用 next 指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表连接起来，这就解决了键冲突的问题。因为 dictEntry 节点组成的链表没有指向链表表尾的指针，所以为了速度考虑，程序总是将新节点添加到链表的表头位置。

### rehash

为了让哈希表的负载因子维持在一个合理的范围之内，当哈希表保存的健值对数量过多或过少时，需要对哈希表进行扩展或者收缩。

- rehash 步骤

  - 为字典的 ht[1]哈希表分配空间
    - 如果执行扩展操作，则 ht[1]的长度大于等于 ht[0].used \* 2 的 2 的 n 次方
    - 如果执行的是收缩操作，则 ht[1]的大小为大于等于 ht[0].used 的 2 的 n 次方。
  - 将 ht[0]的健值对 rehash 到 ht[1]上面。
  - ht[0]的键值对全部 rehash 到 ht[1]时，释放 ht[0],将 ht[1]设置为 ht[0]，并为 ht[1]新创建一个空的哈希表。

- 哈希表的扩展条件

  - 服务器目前没有再执行 BGSAVE 命令或者 BGREWRITEAOF 命令, 并且哈希表的负载因子大于等于 1
  - 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令, 并且哈希表的负载因子大于等于 5

- 哈希表的收缩条件
  - 当哈希表的负载因子小于 0.1 时，程序自动开始对哈希表执行收缩操作。

### 渐进式 rehash

当哈希表的键值对数量太多时，一次性将所有键值对进行 rehash 会对服务器造成压力，甚至导致服务器停止服务。为了保证服务器性能不受影响，采用多次、渐进式地进行 rehash

- 为 ht[1]分配空间，让字典同时持有 ht[0]和 ht[1]两个哈希表。
- 将 rehashidx 设置为 0，表示 rehash 工作正式开始。
- 在 rehash 期间，每次对字典执行添加、删除、查找或者更新的操作时，都会将所操作的健值对进行 rehash，并将 rehashidx 加 1。当进行更新、删除、查找操作时，都会在 ht[0]和 ht[1]上执行，添加键值对时只对 ht[1]操作，保证 ht[0]只减不增。
- 当 ht[0]所有键值对都完成 rehash 时，将 rehashidx 设置为-1，rehash 结束。
