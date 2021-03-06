## redis数据结构

redis接收到一个键值对操作后，能以微秒级别的速度找到数据，并快速完成操作。
redis为什么能够这么快，
一方面，这是因为它是内存数据库，所有操作都在内存上完成，内存的访问速度本身就很快。
另一方面，这要归功于它的数据结构。
这是因为，键值对是按一定的数据结构来组织的，操作键值对最终就是对数据结构进行增删改查操作。
所以高效的数据结构是 Redis 快速处理数据的基础。

Redis 中 value 的数据类型及其底层数据结构：
- String：简单动态字符串。
- List：双向链表、压缩列表。
- Hash：压缩列表、哈希表。
- Sorted Set：压缩列表、跳表。
- Set：哈希表、整数数组。
String 类型的底层实现只有一种数据结构，也就是简单动态字符串。
而 List、Hash、Set 和 Sorted Set 这四种数据类型，都有两种底层实现结构。
通常情况下，我们会把这四种类型称为集合类型，它们的特点是一个键对应了一个集合的数据。

### 键值数据结构

为了实现从键到值的快速访问，Redis 使用了一个哈希表来保存所有键值对，把这个哈希表称为全局哈希表。
一个哈希表，其实就是一个数组，数组的每个元素称为一个哈希桶。所以，我们常说，一个哈希表是由多个哈希桶组成的。
redis中每个哈希桶保存了entry元素，每个entry元素保存了具体的key和value的指针。

哈希表的最大好处很明显，就是让我们可以用 O(1) 的时间复杂度来快速查找到键值对——我们只需要计算键的哈希值，
就可以知道它所对应的哈希桶位置，然后就可以访问相应的 entry 元素。
这个查找过程主要依赖于哈希计算，和数据量的多少并没有直接关系。
也就是说，不管哈希表里有 10 万个键还是 100 万个键，我们只需要一次计算就能找到相应的键。

但是哈希表存在哈希冲突问题和rehash可能带来的操作阻塞，所以哈希查找随着数据量的增多可能会变慢。

### redis如何解决哈希表变慢

哈希冲突：两个不同的key通过哈希运算得到了相同的哈希值，那么两个不同的key就会落在相同的哈希桶上，这就是哈希冲突。
redis通过哈希链表来解决哈希冲突，也就是同一个哈希桶中的不同key通过指针相连，形成一个链表。

随着数据增多，哈希冲突增多，哈希链表也可能越来越长，会影响整个链表的查询效率。这个时候Redis 会对哈希表做 rehash 操作。
rehash 也就是增加现有的哈希桶数量，让逐渐增多的 entry 元素能在更多的桶之间分散保存，减少单个桶中的元素数量，从而减少单个桶中的冲突。

为了使 rehash 操作更高效，Redis 默认使用了两个全局哈希表：哈希表 1 和哈希表 2。
一开始，当你刚插入数据时，默认使用哈希表 1，此时的哈希表 2 并没有被分配空间。
随着数据逐步增多，Redis 开始执行 rehash，这个过程分为三步：
1. 给哈希表 2 分配更大的空间，例如是当前哈希表 1 大小的两倍；
2. 把哈希表 1 中的数据重新进行哈希运算并存贮到哈希表 2 中；
3. 释放哈希表 1 的空间。
到此，我们就可以从哈希表 1 切换到哈希表 2，用增大的哈希表 2 保存更多数据，而原来的哈希表 1 留作下一次 rehash 扩容备用。

这个过程看似简单，但是第二步涉及大量的数据迁移，如果一次性把哈希表 1 中的数据都迁移完，会造成 Redis 线程阻塞，
无法服务其他请求。此时，Redis 就无法快速访问数据了。

redis为了解决这个问题采用了渐进式rehash：
简单来说就是在第二步数据迁移时，Redis 仍然正常处理客户端请求，
每处理一个请求时，从哈希表 1 中的第一个索引位置开始，顺带着将这个索引位置上的所有 entries 迁移到哈希表 2 中；
等处理下一个请求时，再顺带迁移哈希表 1 中的下一个索引位置的 entries。
这样就巧妙地把一次性大量数据迁移的开销，分摊到了多次处理请求的过程中，避免了耗时操作，保证了数据的快速访问。

渐进式rehash执行时，除了根据键值对的操作来进行数据迁移，Redis本身还会有一个定时任务在执行rehash，如果没有键值对操作时，
这个定时任务会周期性地（例如每100ms一次）搬移一些数据到新的哈希表中，这样可以缩短整个rehash的过程。

### 数据操作效率

对于 String 类型来说，找到哈希桶就能直接增删改查了，所以，哈希表的 O(1) 操作复杂度也就是它的复杂度了。

但是，对于集合类型来说，即使找到哈希桶了，还要在集合中再进一步操作。
和 String 类型不同，一个集合类型的值，第一步是通过全局哈希表找到对应的哈希桶位置，第二步是在集合中再增删改查。
那么，集合的操作效率和哪些因素相关呢？
- 首先，与集合的底层数据结构有关。例如，使用哈希表实现的集合，要比使用链表实现的集合访问效率更高。
- 其次，操作效率和这些操作本身的执行特点有关，比如读写一个元素的操作要比读写所有元素的效率高。

### 数据结构复杂度

集合类型的底层数据结构主要有 5 种：整数数组、双向链表、哈希表、压缩列表和跳表。

哈希表的操作特点我们刚刚已经学过了；整数数组和双向链表也很常见，它们的操作特征都是顺序读写，
也就是通过数组下标或者链表的指针逐个元素访问，操作复杂度基本是 O(N)，操作效率比较低。

压缩列表：
压缩列表实际上类似于一个数组，数组中的每一个元素都对应保存一个数据。和数组不同的是，
压缩列表在表头有三个字段 zlbytes、zltail 和 zllen，压缩列表在表尾还有一个 zlend：
- zlbytes ：4字节，记录整个压缩列表占用的内存字节数。在对压缩列表进行内存重分配或者计算 zlend 的位置时使用。
- zltail ：4字节，记录压缩列表表尾节点距离压缩列表的起始地址有多少个字节。通过这个偏移量，无须便利整个压缩列表就可以确定表尾节点的地址。
- zllen ：记录了压缩列表包含的节点数量。当这个属性的值小于 65535 时，这个属性的值就是压缩列表包含节点的数量，
当这个值等于 65535 时，节点的真实数量需要遍历才能计算得出。
- entryX ：压缩列表包含的各个节点，节点的长度由节点保存的内存决定。
- zlend ：特殊值 0xFF（十进制 255），用于标记压缩列表的末端。
在压缩列表中，如果我们要查找定位第一个元素和最后一个元素，可以通过表头三个字段的长度直接定位，复杂度是 O(1)。
而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是 O(N) 了。

Redis的List底层使用压缩列表本质上是将所有元素紧挨着存储，所以分配的是一块连续的内存空间，虽然数据结构本身没有时间复杂度的优势，
但是这样节省空间而且也能避免一些内存碎片；

跳表：
跳表是用于有序元素序列快速搜索查找的一个数据结构，跳表是一个随机化的数据结构，实质就是一种可以进行二分查找的有序链表。
跳表在原有的有序链表上面增加了多级索引，通过索引来实现快速查找。跳表不仅能提高搜索性能，同时也可以提高插入和删除操作的性能。
它在性能上和红黑树，AVL树不相上下，但是跳表的原理非常简单，实现也比红黑树简单很多。
跳表是让链表拥有近乎的接近二分查找的效率的一种数据结构，其原理是给上面加若干层索引，优化查找速度。
当数据量很大时，跳表的查找复杂度是 O(logN)。

### 不同操作的复杂度

集合类型的操作类型很多，有读写单个集合元素的，例如 HGET、HSET，也有操作多个元素的，例如 SADD，还有对整个集合进行遍历操作的，
例如 SMEMBERS。这么多操作，它们的复杂度也各不相同。而复杂度的高低又是我们选择集合类型的重要依据。

1. 单元素操作

单元素操作，是指每一种集合类型对单个数据实现的增删改查操作。
例如，Hash 类型的 HGET、HSET 和 HDEL，Set 类型的 SADD、SREM、SRANDMEMBER 等。
这些操作的复杂度由集合采用的数据结构决定，例如，HGET、HSET 和 HDEL 是对哈希表做操作，所以它们的复杂度都是 O(1)；
Set 类型用哈希表作为底层数据结构时，它的 SADD、SREM、SRANDMEMBER 复杂度也是 O(1)。

这里，有个地方你需要注意一下，集合类型支持同时对多个元素进行增删改查，
例如 Hash 类型的 HMGET 和 HMSET，Set 类型的 SADD 也支持同时增加多个元素。
此时，这些操作的复杂度，就是由单个元素操作复杂度和元素个数决定的。例如，HMSET 增加 M 个元素时，复杂度就从 O(1) 变成 O(M) 了。

2. 范围操作

范围操作，是指集合类型中的遍历操作，可以返回集合中的所有数据，比如 Hash 类型的 HGETALL 和 Set 类型的 SMEMBERS，
或者返回一个范围内的部分数据，比如 List 类型的 LRANGE 和 ZSet 类型的 ZRANGE。这类操作的复杂度一般是 O(N)，比较耗时，我们应该尽量避免。

Redis 从 2.8 版本开始提供了 SCAN 系列操作（包括 HSCAN，SSCAN 和 ZSCAN），这类操作实现了渐进式遍历，每次只返回有限数量的数据。
这样一来，相比于 HGETALL、SMEMBERS 这类操作来说，就避免了一次性返回所有元素而导致的 Redis 阻塞。

3. 统计操作

统计操作，是指集合类型对集合中所有元素个数的记录，例如 LLEN 和 SCARD。这类操作复杂度只有 O(1)，
这是因为当集合类型采用压缩列表、双向链表、整数数组这些数据结构时，这些结构中专门记录了元素的个数统计，因此可以高效地完成相关操作。

4. 例外情况

例外情况，是指某些数据结构的特殊记录，例如压缩列表和双向链表都会记录表头和表尾的偏移量。
这样一来，对于 List 类型的 LPOP、RPOP、LPUSH、RPUSH 这四个操作来说，它们是在列表的头尾增删元素，这就可以通过偏移量直接定位，
所以它们的复杂度也只有 O(1)，可以实现快速操作。

### 总结

Redis 之所以能快速操作键值对，一方面是因为 O(1) 复杂度的哈希表被广泛使用，包括 String、Hash 和 Set，它们的操作复杂度基本由哈希表决定，
另一方面，Sorted Set 也采用了 O(logN) 复杂度的跳表。不过，集合类型的范围操作，因为要遍历底层数据结构，复杂度通常是 O(N)。
这里，建议是：用其他命令来替代，例如可以用 SCAN 来代替，避免在 Redis 内部产生费时的全集合遍历操作。

当然，我们不能忘了复杂度较高的 List 类型，它的两种底层实现结构：双向链表和压缩列表的操作复杂度都是 O(N)。
因此，建议是：因地制宜地使用 List 类型。例如，既然它的 POP/PUSH 效率很高，那么就将它主要用于 FIFO 队列场景，而不是作为一个可以随机读写的集合。

整数数组和压缩列表在查找时间复杂度方面并没有很大的优势，Redis 还会把它们作为底层数据结构原因主要是内存利用率，
数组和压缩列表都是非常紧凑的数据结构，它比链表占用的内存要更少。Redis是内存数据库，大量数据存到内存中，此时需要做尽可能的优化，提高内存的利用率。