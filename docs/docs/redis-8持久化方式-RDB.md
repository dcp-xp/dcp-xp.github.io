在运行情况下， Redis 以数据结构的形式将数据维持在内存中， 为了让这些数据在 Redis 重启之后仍然可用， Redis 分别提供了 RDB 和 AOF 两种持久化模式。

在 Redis 运行时， RDB 程序将当前内存中的数据库快照保存到磁盘文件中， 在 Redis 重启动时， RDB 程序可以通过载入 RDB 文件来还原数据库的状态。

RDB 功能最核心的是 rdbSave 和 rdbLoad 两个函数，前者用于生成 RDB 文件到磁盘， 而后者则用于将 RDB 文件中的数据重新载入到内存中：

![RDB内存与磁盘交互](../images/RDB内存与磁盘交互.png)

本章先介绍 SAVE 和 BGSAVE 命令的实现， 以及 rdbSave 和 rdbLoad 两个函数的运行机制， 然后以图表的方式， 分部分来介绍 RDB 文件的组织形式。

## 保存

rdbSave 函数负责将内存中的数据库数据以 RDB 格式保存到磁盘中， 如果 RDB 文件已存在，那么新的 RDB 文件将替换已有的 RDB 文件。

在保存 RDB 文件期间，主进程会被阻塞，直到保存完成为止。

SAVE 和 BGSAVE 两个命令都会调用 rdbSave 函数，但它们调用的方式各有不同：

- SAVE 直接调用 rdbSave ，阻塞 Redis 主进程，直到保存完成为止。在主进程阻塞期间，服务器不能处理客户端的任何请求。
- BGSAVE 则 fork 出一个子进程，子进程负责调用 rdbSave ，并在保存完成之后向主进程发送信号，通知保存已完成。因为 rdbSave 在子进程被调用，所以 Redis 服务器在 BGSAVE 执行期间仍然可以继续处理客户端的请求。

通过伪代码来描述这两个命令，可以很容易地看出它们之间的区别：

```
def SAVE():
    rdbSave()

def BGSAVE():
    pid = fork()
    if pid == 0:
        # 子进程保存 RDB
        rdbSave()
    elif pid > 0:
        # 父进程继续处理请求，并等待子进程的完成信号
        handle_request()
    else:
        # pid == -1
        # 处理 fork 错误
        handle_fork_error()
```

## SAVE 、 BGSAVE 、 AOF 写入和 BGREWRITEAOF

### SAVE

当 SAVE 执行时，Redis 服务器是阻塞的，所以当 SAVE 正在执行时， 新的 SAVE、BGSAVE 或 BGREWRITEAOF 调用都不会产生任何作用。

只有在上一个 SAVE 执行完毕、 Redis 重新开始接受请求之后， 新的 SAVE、BGSAVE 或 BGREWRITEAOF 命令才会被处理。

另外， 因为 AOF 写入由后台线程完成， 而 BGREWRITEAOF 则由子进程完成， 所以在 SAVE 执行的过程中， AOF 写入和 BGREWRITEAOF 可以同时进行。

### BGSAVE

在执行 SAVE 命令之前， 服务器会检查 BGSAVE 是否正在执行当中，如果是的话，服务器就不调用 rdbSave ，而是向客户端返回一个出错信息，告知在 BGSAVE 执行期间，不能执行 SAVE 。

这样做可以避免 SAVE 和 BGSAVE 调用的两个 rdbSave 交叉执行，造成竞争条件。

另一方面， 当 BGSAVE 正在执行时， 调用新 BGSAVE 命令的客户端会收到一个出错信息， 告知 BGSAVE 已经在执行当中。

BGREWRITEAOF 和 BGSAVE 不能同时执行：

- 如果 BGSAVE 正在执行，那么 BGREWRITEAOF 的重写请求会被延迟到 BGSAVE 执行完毕之后进行，执行 BGREWRITEAOF 命令的客户端会收到请求被延迟的回复。
- 如果 BGREWRITEAOF 正在执行，那么调用 BGSAVE 的客户端将收到出错信息，表示这两个命令不能同时执行。

BGREWRITEAOF 和 BGSAVE 两个命令在操作方面并没有什么冲突的地方， 不能同时执行它们只是一个性能方面的考虑： 并发出两个子进程， 并且两个子进程都同时进行大量的磁盘写入操作， 这怎么想都不会是一个好主意。

## 载入

当 Redis 服务器启动时， rdbLoad 函数就会被执行， 它读取 RDB 文件， 并将文件中的数据库数据载入到内存中。

在载入期间， 服务器每载入 1000 个键就处理一次所有已到达的请求， 不过只有 PUBLISH 、 SUBSCRIBE 、 PSUBSCRIBE 、 UNSUBSCRIBE 、 PUNSUBSCRIBE 五个命令的请求会被正确地处理， 其他命令一律返回错误。 等到载入完成之后， 服务器才会开始正常处理所有命令。

发布与订阅功能和其他数据库功能是完全隔离的，前者不写入也不读取数据库，所以在服务器载入期间，订阅与发布功能仍然可以正常使用，而不必担心对载入数据的完整性产生影响。
另外， 因为 AOF 文件的保存频率通常要高于 RDB 文件保存的频率， 所以一般来说， AOF 文件中的数据会比 RDB 文件中的数据要新。

因此， 如果服务器在启动时， 打开了 AOF 功能， 那么程序优先使用 AOF 文件来还原数据。 只有在 AOF 功能未打开的情况下， Redis 才会使用 RDB 文件来还原数据。

RDB 文件结构
前面介绍了保存和读取 RDB 文件的两个函数，现在，是时候介绍 RDB 文件本身了。

一个 RDB 文件可以分为以下几个部分：

![RDB文件结构图](../images/RDB文件结构图.png)

以下的几个小节将分别对这几个部分的保存和读入规则进行介绍。

### REDIS

文件的最开头保存着 REDIS 五个字符，标识着一个 RDB 文件的开始。

在读入文件的时候，程序可以通过检查一个文件的前五个字节，来快速地判断该文件是否有可能是 RDB 文件。

### RDB-VERSION

一个四字节长的以字符表示的整数，记录了该文件所使用的 RDB 版本号。

目前的 RDB 文件版本为 0006 。

因为不同版本的 RDB 文件互不兼容，所以在读入程序时，需要根据版本来选择不同的读入方式。

### DB-DATA

这个部分在一个 RDB 文件中会出现任意多次，每个 DB-DATA 部分保存着服务器上一个非空数据库的所有数据。

### SELECT-DB

这域保存着跟在后面的键值对所属的数据库号码。

在读入 RDB 文件时，程序会根据这个域的值来切换数据库，确保数据被还原到正确的数据库上。

### EOF

标志着数据库内容的结尾（不是文件的结尾），值为 rdb.h/EDIS_RDB_OPCODE_EOF （255）。

### CHECK-SUM

RDB 文件所有内容的校验和， 一个 uint_64t 类型值。

REDIS 在写入 RDB 文件时将校验和保存在 RDB 文件的末尾， 当读取时， 根据它的值对内容进行校验。

如果这个域的值为 0 ， 那么表示 Redis 关闭了校验和功能。
