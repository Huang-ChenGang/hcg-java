## 键值数据库

我们知道redis是一个键值数据库，所以了解redis之前先了解一下键值数据库。
先知道两个概念：
- 数据模型：在数据库里可以存什么样的数据。
- 操作接口：可以对数据做什么操作。
键值数据库只提供简单的操作接口，不支持复杂的聚合运算。

### 数据模型

既然是键值数据库，那数据模型就是key-value模型。
redis中，key是String类型，而value有string、list、hash、set、zset(sorted-set)五种类型。
Redis 能够在实际业务场景中得到广泛的应用，就是得益于支持多样化类型的 value。

不同 value 类型的实现，不仅可以支撑不同业务的数据需求，
而且也隐含着不同数据结构在性能、空间效率等方面的差异，
从而导致不同的 value 操作之间存在着差异。

### 操作接口

一个数据库的基本功能就是增删改查，那键值数据库必须要支持的基本操作有以下几个：
- put/set：新增或更新一个key-value对。
- get：根据key查询对应的value。
- delete：根据key删除整个key-value对。
- scan：查询一个用户在一段时间内的访问记录。

### 数据是否保存在内存

- 保存在内存：数据读写快，有数据丢失风险。
- 保存在外存：避免了数据丢失，但是读写慢，整体性能低。

用于缓存的数据库要求读写速度快，允许数据丢失，所以选择保存在内存。
redis是主要用于缓存的数据库，所以redis的数据时保存在内存的，但是提供了持久化方案。

### 键值数据库的内部架构

一个键值数据库包括访问框架、索引模块、操作模块、存储模块四个部分。

访问框架：
访问框架就是访问键值数据库的方式，一般有以下两种：
- 通过函数库调用的方式供外部应用使用。
- 通过网络框架以Socket通信的形式对外提供键值对操作，包括SocketServer和协议解析。
redis采用的是后者，也就是通过网络框架来访问。

索引模块：
索引模块的作用是用来定义键值对的位置，也就是找到键值对存在哪里。
索引的作用就是让键值数据库根据key找到相应的value的存储位置，从而执行相应的操作。
索引的类型有很多，常见的有哈希表、B+ 树、字典树等。不同的索引结构在性能、空间消耗、并发控制等方面具有不同的特征。
一般而言，内存键值数据库（例如 Redis）采用哈希表作为索引，很大一部分原因在于，其键值数据基本都是保存在内存中的，
而内存的高性能随机访问特性可以很好地与哈希表 O(1) 的操作复杂度相匹配。
对于 Redis 而言，它的 value 支持多种类型，当我们通过索引找到一个 key 所对应的 value 后，
仍然需要从 value 的复杂结构（例如集合和列表）中进一步找到我们实际需要的数据，这个操作的效率本身就依赖于它们的实现结构。
Redis 采用一些常见的高效索引结构作为某些 value 类型的底层数据结构，这一技术路线为 Redis 实现高性能访问提供了良好的支撑。

操作模块：
操作模块是指键值数据库提供的操作逻辑，如put/set、get、delete、scan等。

存储模块：
顾名思义就是数据的存储，也就是持久化功能。为了保证重启后数据不丢失并且能够继续提供服务，键值数据库需要支持数据持久化。
redis提供了持久化功能和优化机制，之后再看。