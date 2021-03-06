## AOF (Append Only File) 日志持久化

Redis 是内存数据库，如果不设置持久化策略，一旦服务器宕机，内存中的数据将全部丢失。
我们很容易想到的一个解决方案是，从后端数据库恢复这些数据，但这种方式存在两个问题：
一是，需要频繁访问数据库，会给数据库带来巨大的压力；
二是，这些数据是从慢速数据库中读取出来的，性能肯定比不上从 Redis 中读取，导致使用这些数据的应用程序响应变慢。
所以，对 Redis 来说，实现数据的持久化，避免从后端数据库中进行恢复，是至关重要的。

### AOF 是什么

AOF 是以日志的方式对 Redis 进行持久化存储。AOF 是后写日志，意思就是 Redis 先执行命令，把数据写入内存，然后才记录日志。

那 AOF 为什么要先执行命令再记日志呢？要回答这个问题，我们要先知道 AOF 里记录了什么内容。
传统数据库的日志，例如 redo log（重做日志），记录的是修改后的数据，而 AOF 里记录的是 Redis 收到的每一条命令，这些命令是以文本形式保存的。

我们以 Redis 收到“set testkey testvalue”命令后记录的日志为例，看看 AOF 日志的内容。
其中，“*3”表示当前命令有三个部分，每部分都是由“$+数字”开头，后面紧跟着具体的命令、键或值。
这里，“数字”表示这部分中的命令、键或值一共有多少字节。例如，“$3 set”表示这部分有 3 个字节，也就是“set”命令。
示例图：

![AOF日志内容](https://static001.geekbang.org/resource/image/4d/9f/4d120bee623642e75fdf1c0700623a9f.jpg)

但是，为了避免额外的检查开销，Redis 在向 AOF 里面记录日志的时候，并不会先去对这些命令进行语法检查。
所以，如果先记日志再执行命令的话，日志中就有可能记录了错误的命令，Redis 在使用日志恢复数据时，就可能会出错。
而写后日志这种方式，就是先让系统执行命令，只有命令能执行成功，才会被记录到日志中，否则，系统就会直接向客户端报错。
所以，Redis 使用写后日志这一方式的一大好处是，可以避免出现记录错误命令的情况。
除此之外，AOF 还有一个好处：它是在命令执行后才记录日志，所以不会阻塞当前的写操作。

不过 AOF 有两个潜在的风险：
- 如果刚执行完一个命令，还没有来得及记日志就宕机了，那么这个命令和相应的数据就有丢失的风险。
如果此时 Redis 是用作缓存，还可以从后端数据库重新读入数据进行恢复，
但是，如果 Redis 是直接用作数据库的话，此时，因为命令没有记入日志，所以就无法用日志进行恢复了。
- AOF 虽然避免了对当前命令的阻塞，但可能会给下一个操作带来阻塞风险。
这是因为，AOF 日志也是在主线程中执行的，如果在把日志文件写入磁盘时，磁盘写压力大，就会导致写盘很慢，进而导致后续的操作也无法执行了。

仔细分析的话，你就会发现，这两个风险都是和 AOF 写回磁盘的时机相关的。
这也就意味着，如果我们能够控制一个写命令执行完后 AOF 日志写回磁盘的时机，这两个风险就解除了。

### AOF 写回策略

针对写回磁盘的时机，AOF 机制给我们提供了三个选择，也就是 AOF 配置项 appendfsync 的三个可选值：
- Always：同步写回：每个写命令执行完，立马同步地将日志写回磁盘。
- Everysec：每秒写回：每个写命令执行完，只是先把日志写到 AOF 文件的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘。
- No：操作系统控制的写回：每个写命令执行完，只是先把日志写到 AOF 文件的内存缓冲区，由操作系统决定何时将缓冲区内容写回磁盘。

针对避免主线程阻塞和减少数据丢失问题，这三种写回策略都无法做到两全其美。我们来分析下其中的原因：
- Always：同步写回：可以做到基本不丢数据，但是它在每一个写命令后都有一个慢速的落盘操作，不可避免地会影响主线程性能。
- Everysec：每秒写回：采用一秒写回一次的频率，避免了“同步写回”的性能开销，虽然减少了对系统性能的影响，
但是如果发生宕机，上一秒内未落盘的命令操作仍然会丢失。所以，这只能算是，在避免影响主线程性能和避免数据丢失两者间取了个折中。
- No：操作系统控制的写回：性能好，Redis 在写完缓冲区后，就可以继续执行后续的命令。
但是落盘的时机已经不在 Redis 手中了，只要 AOF 记录没有写回磁盘，一旦宕机对应的数据就丢失了。

总结一下就是：想要获得高性能，就选择 No 策略；
如果想要得到高可靠性保证，就选择 Always 策略；
如果允许数据有一点丢失，又希望性能别受太大影响的话，那么就选择 Everysec 策略。

三种写回策略体现了系统设计中的一个重要原则 ，即 trade-off，或者称为“取舍”，指的就是在性能和可靠性保证之间做取舍。
这是做系统设计和开发的一个关键哲学，希望能充分地理解这个原则，并在日常开发中加以应用。

AOF 是以文件的形式在记录接收到的所有写命令。随着接收的写命令越来越多，AOF 文件会越来越大。
这也就意味着，我们一定要小心 AOF 文件过大带来的性能问题：
- 文件系统本身对文件大小有限制，无法保存过大的文件。
- 如果文件太大，之后再往里面追加命令记录的话，效率也会变低。
- 如果发生宕机，AOF 中记录的命令要一个个被重新执行，用于故障恢复，如果日志文件太大，整个恢复过程就会非常缓慢，这就会影响到 Redis 的正常使用。

### AOF 重写机制

为了解决 AOF 文件过大带来的性能问题，Redis 提供了 AOF 重写机制。
AOF 重写机制就是在重写时，Redis 根据数据库的现状创建一个新的 AOF 文件，
也就是说，读取数据库中的所有键值对，然后对每一个键值对用一条命令记录它的写入。
比如说，当读取了键值对“testkey”: “testvalue”之后，重写机制会记录 set testkey testvalue 这条命令。
这样，当需要恢复时，可以重新执行该命令，实现“testkey”: “testvalue”的写入。

AOF 重写机制为什么能让 AOF 文件变小呢？
如果一个键值对被多次操作的话，那么旧的 AOF 文件中就会有这个键值对的多条记录。但是在新的 AOF 文件中只会有一条新增的命令。

AOF 重写机制并没有使用原来的 AOF 日志文件，而是重新生成了一个新的日志文件。两个原因：
- 父子进程写同一个文件必然会产生竞争问题，控制竞争就意味着会影响父进程的性能。
- 如果AOF重写过程中失败了，那么原本的AOF文件相当于被污染了，无法做恢复使用。
所以Redis AOF重写一个新文件，重写失败的话，直接删除这个文件就好了，不会对原先的AOF文件产生影响。等重写完成之后，直接替换旧文件即可。

虽然 AOF 重写后，日志文件会缩小，但是，要把整个数据库的最新数据的操作日志都写回磁盘，仍然是一个非常耗时的过程。
这时，我们就要继续关注另一个问题了：重写会不会阻塞主线程？

### bgrewriteaof 子进程

和 AOF 日志由主线程写回不同，重写过程是由后台子进程 bgrewriteaof 来完成的，这也是为了避免阻塞主线程，导致数据库性能下降。

重写的过程总结为“一个拷贝，两处日志”。

1. 一个拷贝

理解一个拷贝前先了解一下写时复制机制。
写时复制机制（Copy-on-write，简称COW）是一种计算机程序设计领域的优化策略。
其核心思想是，如果有多个调用者（callers）同时要求相同资源（如内存或磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，
直到某个调用者试图修改资源的内容时，系统才会真正复制一份专用副本（private copy）给该调用者，而其他调用者所见到的最初的资源仍然保持不变。
这过程对其他的调用者都是透明的（transparently）。此作法主要的优点是如果调用者没有修改该资源，就不会有副本（private copy）被创建，
因此多个调用者只是读取操作时可以共享同一份资源。

“一个拷贝”就是指，每次执行重写时，主线程 fork 出后台的 bgrewriteaof 子进程。
此时，fork 会把主线程的内存拷贝一份给 bgrewriteaof 子进程，这里面就包含了数据库的最新数据。
然后，bgrewriteaof 子进程就可以在不影响主线程的情况下，逐一把拷贝的数据写成操作，记入重写日志。

这里会有两个阻塞点：
- fork子进程时。fork这个瞬间一定是会阻塞主线程的，fork采用操作系统提供的写时复制(Copy On Write)机制，
就是为了避免一次性拷贝大量内存数据给子进程造成的长时间阻塞问题，但fork子进程需要拷贝进程必要的数据结构，
其中有一项就是拷贝内存页表（虚拟内存和物理内存的映射索引表），这个拷贝过程会消耗大量CPU资源，拷贝完成之前整个进程是会阻塞的，
阻塞时间取决于整个实例的内存大小，实例越大，内存页表越大，fork阻塞时间越久。
拷贝内存页表完成后，子进程与父进程指向相同的内存地址空间，也就是说此时虽然产生了子进程，但是并没有申请与父进程相同的内存大小。
那什么时候父子进程才会真正内存分离呢？“写时复制”顾名思义，就是在写发生时，才真正拷贝内存真正的数据。
- AOF重写过程中父进程产生写入时。fork出的子进程指向与父进程相同的内存地址空间，此时子进程就可以执行AOF重写，
把内存中的所有数据写入到AOF文件中。但是此时父进程依旧是会有流量写入的，如果父进程操作的是一个已经存在的key，
那么这个时候父进程就会真正拷贝这个key对应的内存数据，申请新的内存空间，这样逐渐地，父子进程内存数据开始分离，
父子进程逐渐拥有各自独立的内存空间。因为内存分配是以页为单位进行分配的，默认4k，如果父进程此时操作的是一个bigKey，
重新申请大块内存耗时会变长，可能会产阻塞风险。另外，如果操作系统开启了内存大页机制(Huge Page，页面大小2M)，
那么父进程申请内存时阻塞的概率将会大大提高，所以在Redis机器上需要关闭Huge Page机制。
Redis每次fork生成RDB或AOF重写完成后，都可以在Redis log中看到父进程重新申请了多大的内存空间。

fork后，父进程修改数据采用写时复制，复制的粒度为一个内存页。如果只是修改一个256B的数据，父进程需要读原来的内存页，
然后再映射到新的物理地址写入。一读一写会造成读写放大。如果内存页越大（例如2MB的大页），那么读写放大也就越严重，对Redis性能造成影响。
Huge page机制在实际使用Redis时是建议关掉的。

2. 两处日志

因为主线程未阻塞，仍然可以处理新来的操作。此时，如果有写操作，第一处日志就是指正在使用的 AOF 日志，Redis 会把这个操作写到它的缓冲区。
这样一来，即使宕机了，这个 AOF 日志的操作仍然是齐全的，可以用于恢复。

而第二处日志，就是指新的 AOF 重写日志。这个操作也会被写到重写日志的缓冲区。这样，重写日志也不会丢失最新的操作。
等到拷贝数据的所有操作记录重写完成后，重写日志记录的这些最新操作也会写入新的 AOF 文件，以保证数据库最新状态的记录。
此时，我们就可以用新的 AOF 文件替代旧文件了。

AOF 重写流程图：

![AOF重写](https://static001.geekbang.org/resource/image/6b/e8/6b054eb1aed0734bd81ddab9a31d0be8.jpg)

总结来说，每次 AOF 重写时，Redis 会先执行一个内存拷贝，用于重写；然后，使用两个日志保证在重写过程中，新写入的数据不会丢失。
而且，因为 Redis 采用额外的子进程进行数据重写，所以，这个过程并不会阻塞主线程。

### AOF 重写机制触发

1. 手动触发

为了减小aof文件的体量，可以手动发送“bgrewriteaof”指令，通过子进程生成更小体积的aof，然后替换掉旧的、大体量的aof文件。

2. 自动触发

配置自动触发：
- auto-aof-rewrite-min-size 64mb
- auto-aof-rewrite-percentage 100
这两个配置项的意思是，在aof文件体量超过64mb，且比上次重写后的体量增加了100%时自动触发重写。我们可以修改这些参数达到自己的实际要求。

自动触发配置解释：
- auto-aof-rewrite-min-size: 表示运行AOF重写时文件的最小大小，默认为64MB
- auto-aof-rewrite-percentage: 这个值的计算方法是：当前AOF文件大小和上一次重写后AOF文件大小的差值，再除以上一次重写后AOF文件大小。
也就是当前AOF文件比上一次重写后AOF文件的增量大小，和上一次重写后AOF文件大小的比值。

AOF文件大小同时超出上面这两个配置项时，会触发AOF重写。

### AOF 配置

Redis 默认是关闭 AOF 持久化模式：appendonly no，要开启把no修改成YES。

appendfilename "appendonly.aof"，持久化的文件名称。

no-appendfsync-on-rewrite no，默认为 no，表示是否允许在 AOF 重写期间执行写回策略。
Redis在后台写RDB文件或重写AOF文件期间会存在大量磁盘IO，此时调用fsync可能会阻塞。

### 问题

写回策略和重写机制都是在"写日志"这一过程中发挥作用的。例如，写回策略的选择可以避免记日志时阻塞主线程，重写可以避免日志文件过大。
但是，在"读日志"的过程中，也就是使用 AOF 进行故障恢复时，我们仍然需要把所有的操作记录都运行一遍。
再加上 Redis 的单线程设计，这些命令操作只能一条一条按顺序执行，这个"重放"的过程就会很慢了。