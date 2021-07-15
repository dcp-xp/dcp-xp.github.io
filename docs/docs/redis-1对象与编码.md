# Redis 基础系列（一）—— Redis 对象与编码

Redis 使用对象来表示数据库中的健值对，当向 Redis 数据库中设置一个 key-value 时，数据库至少会创建两个对象，一个对象用作键值对的键（键对象），一个对象用作键值对的值（值对象）。Redis 中的每个对象都由一个 redisObject 结构表示，该结构中和保存数据有关的三个属性分别是 type 属性，encoding 属性和 ptr 属性。还有用于对象引用计数的 refcount 属性，记录对象空转时长的 lru 属性。

```C
typedef struct redisObject {
    unsigned type:4;       // 类型
    unsigned encoding:4;   // 编码
    unsigned lru:LRU_BITS; // 记录对象空转时长
    int refcount;          // 引用计数
    void *ptr;             // 指向底层实现数据结构的指针
} robj;
```

```Shell
# 查看命令
debug object key
```

## 类型

```Shell
# 查看命令
type key
```

对象的 type 属性记录对象的类型，Redis 中共有 5 中对象类型：字符串对象、列表对象、哈希对象、集合对象、有序集合对象。在 Redis 的键值对中，键总是一个字符串对象，而值可以是 5 种对象类型中的任何一种。

| 对象         | 类型常量     | type 命令的输出 |
| ------------ | ------------ | --------------- |
| 字符串对象   | REDIS_STRING | string          |
| 列表对象     | REDIS_LIST   | list            |
| 哈希对象     | REDIS_HASH   | hash            |
| 集合对象     | REDIS_SET    | set             |
| 有序集合对象 | REDIS_ZSET   | zset            |

## 编码

```shell
#查看命令
object encoding key
```

> redisObject 结构中的 encoding 属性记录着对象的编码方式，编码的方式决定着 ptr 指针指向的底层实现数据结构。Redis 中的对象都有两种或以上的编码方式，每一种编码都有对应数据结构。

| 类型         | 编码常量               | 编码所对应的底层数据结构    | OBJECT ENCODING 命令输出 | 对象                                             |
| ------------ | ---------------------- | --------------------------- | ----------------- | ------------------------------------------------ |
| REDIS_STRING | OBJ_ENCODING_INT       | long 类型的整数             | int               | 使用整数值实现的字符串对象                       |
| REDIS_STRING | OBJ_ENCODING_EMBSTR    | embstr 编码的简单动态字符串 | embstr            | 使用 embstr 编码的简单动态字符串实现的字符串对象 |
| REDIS_STRING | OBJ_ENCODING_RAM       | 简单动态字符串              | raw               | 使用简单动态字符串实现的字符串对象               |
| REDIS_LIST   | OBJ_ENCODING_QUICKLIST | 快表                        | linkedlist        | 使用压缩列表实现的列表对象                       |
| REDIS_HASH   | OBJ_ENCODING_ZIPLIST   | 压缩表                      | ziplist           | 使用压缩列表实现哈希对象                         |
| REDIS_HASH   | OBJ_ENCODING_HT        | 字典                        | hashtable         | 使用字典实现的哈希对象                           |
| REDIS_SET    | OBJ_ENCODING_INTSET    | 整数集合                    | intset            | 使用整数集合实现的集合对象                       |
| REDIS_SET    | OBJ_ENCODING_HT        | 字典                        | hashtable         | 使用字典实现的集合对象                           |
| REDIS_ZSET   | OBJ_ENCODING_ZIPLIST   | 压缩表                      | ziplist           | 使用压缩列表实现的有序集合对象                   |

## 类型检查与编码检查

Redis 中操作键的命令基本上可以分为两种类型：一种命令可以对任何类型的键执行，如 DEL 命令、TYPE 命令；另一种只针对特定的对象类型执行，如 GET 命令只针对 String 对象，如果对 List 对象使用则会报类型错误。

服务器接收到一条命令时，会对其进行类型检查；还会根据其编码方式选择正确的实现代码来执行命令。类型检查根据 redisObject 结构的 type 属性进行，编码检查根据 encoding 属性进行。类型检查和编码检查都是实现多态命令的方式，前者是基于类型的多态，后者是基于编码的多态。

## 内存回收与对象共享

对象的整个生命周期可以划分为创建对象、操作对象、释放对象三个阶段。Redis 底层实现使用 C 语言，而 C 语言并不具备自动内存回收功能，所以 Redis 在自己的对象系统中使用引用计数技术实现内存回收机制。对象的引用计数信息会存于 redisObject 的 refcount 属性中，对象的引用计数信息会随着对象的使用状态而不断变化。

- 创建新对象时，引用计数的值会被初始化为 1；
- 当对象被一个新程序使用时，它的引用计数值会被增一；
- 当对象不再被一个程序使用时，它的引用计数会被减一；
- 当对象的引用计数变为 0 时，对象所占用的内存会被释放。

除了用于实现引用计数回收内存之外，refcount 属性还被用于 REDIS_ENCODING_INT 编码的字符串对象的共享。当创建一个值为整数值类型的键值对时，数据库会先检查是否存在一个值相等的字符串对象，存在则将键指向已存在的字符串对象，并将对象的 refcount 加一。

Redis 在初始化服务器时，会创建一万个字符串对象，包含 0~9999 的所有整数值，当服务器需要用到这些值时，服务器就会使用这些共享对象。

Redis 不共享包含字符串的对象，因为验证字符串是否完全相同的复杂度太高，会占用太多的 CPU 资源。

## 对象的空转时长与数据淘汰策略

redisObject 结构中的 lru 属性记录对象最后操作的时间，使用 OBJECT IDLETIME 命令可以获得指定键的空转时长。空转时长可服务于数据淘汰策略，当数据库占用的内存大于所设置的 maxmemory，并且 maxmomery-policy 设置为 volatile-lru 或 allkeys-lru 时，将对空转时长最高的部分数据进行删除。
Redis 中的淘汰策略

- volatile-lru(least recently used):最近最少使用算法，从设置了过期时间的键 key 中选择空转时间最长的键值对清除掉；
- volatile-lfu(least frequently used):最近最不经常使用算法，从设置了过期时间的键中选择某段时间之内使用频次最小的键值对清除掉；
- volatile-ttl:从设置了过期时间的键中选择过期时间最早的键值对清除；
- volatile-random:从设置了过期时间的键中，随机选择键进行清除；
- allkeys-lru:最近最少使用算法，从所有的键中选择空转时间最长的键值对清除；
- allkeys-lfu:最近最不经常使用算法，从所有的键中选择某段时间之内使用频次最少的键值对清除；
- allkeys-random:所有的键中，随机选择键进行删除；
- noeviction:不做任何的清理工作，在 redis 的内存超过限制之后，所有的写入操作都会返回错误；但是读操作都能正常的进行;
