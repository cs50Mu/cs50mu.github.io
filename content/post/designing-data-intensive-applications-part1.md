+++
title = "designing data intensive applications part1"
date = "2020-11-12"
slug = "2020/11/12/designing-data-intensive-applications-part1"
Categories = ["design", "distributed system"]
+++

## Part I - Foundations of Data Systems

### Chapter 1 - Reliable, Scalable, and Maintainable Applications

> Reliability  / 可用性

The system should continue to work correctly (performing the correct function at the desired level of performance) even in the face of adversity (hardware or soft‐ ware faults, and even human error).

Note that a fault is not the same as a failure [2]. A fault is usually defined as one component of the system deviating from its spec, whereas a failure is when the system as a whole stops providing the required service to the user. 注意区分 fault 和 failure

有哪些种类的falut呢？

- hardware faults：硬盘坏了
- software errors：内存泄漏
- human errors：运维操作失误

> Scalability / 可扩展性

As the system grows (in data volume, traffic volume, or complexity), there should be reasonable ways of dealing with that growth.

Even if a system is working reliably today, that doesn’t mean it will necessarily **work reliably in the future**. One common reason for degradation is **increased load.**

Scalability is the term we use to describe **a system’s ability to cope with increased load.**

Note, however, that it is not a one-dimensional label that we can attach to a system: **it is meaningless to say “X is scalable” or “Y doesn’t scale.”** Rather, discussing scalability means considering questions like “If the system grows in a particular way, what are our options for coping with the growth?” and “How can we add computing resources to handle the additional load?”

**Describing Load**

First, we need to succinctly describe the current load on the system; only then can we discuss growth questions. 首先要明确现状

**Describing Performance**

You can look at it in two ways:

- When you increase a load parameter and keep the system resources (CPU, mem‐ ory, network bandwidth, etc.) unchanged, how is the performance of your system affected?
- When you increase a load parameter, how much do you need to increase the resources if you want to keep performance unchanged?

**percentile**：For example, if the 95th percentile response time is 1.5 seconds, that means 95 out of 100 requests take less than 1.5 seconds, and 5 out of 100 requests take 1.5 seconds or more.

**Approaches for Coping with Load**

An architecture that is appropriate for one level of load is unlikely to cope with 10 times that load. 

- vertical scaling, moving to a more powerful machine
- horizontal scaling, distributing the load across multiple smaller machines

An architecture that scales well for a particular application is built around assumptions of which operations will be common and which will be rare. 没有万能的架构，都是根据场景而来的，不能照搬

In an early-stage startup or an unpro‐ ven product it’s usually more important to be able to iterate quickly on product features than it is to scale to some hypothetical future load. 前期先重点关注feature

> Maintainability / 可维护性

Over time, many different people will work on the system (engineering and oper‐ ations, both maintaining current behavior and adapting the system to new use cases), and they should all be able to work on it **productively**

**Simplicity: Managing Complexity**

accidental complexity: we define complexity as accidental if it is not inherent in the problem that the software solves (as seen by the users) but arises only from the implementation.


### CHAPTER 2 - Data Models and Query Languages

each layer hides the complexity of the layers below it by providing a clean data model.

**The Object-Relational Mismatch**

Most application development today is done in object-oriented programming lan‐ guages, which leads to a common criticism of the SQL data model: if data is stored in relational tables, an awkward translation layer is required between the objects in the application code and the database model of tables, rows, and columns. The discon‐ nect between the models is sometimes called an impedance mismatch

However, when it comes to representing many-to-one and many-to-many relation‐ ships, relational and document databases are not fundamentally different: in both cases, the related item is referenced by a unique identifier, which is called a foreign key in the relational model and a document reference in the document model [9]. **That identifier is resolved at read time by using a join or follow-up queries.** 还是需要join或者再次查询

如何选择？realational model OR document model？

跟场景有关

If the data in your application has a document-like structure (i.e., a tree of one-to-many relationships, where typically the entire tree is loaded at once), then it’s probably a good idea to use a document model.

However, if your application does use many-to-many relationships, the document model becomes less appealing. It’s possible to reduce the need for joins by denormal‐ izing, but then the application code needs to do additional work to keep the denor‐ malized data consistent. Joins can be emulated in application code by making multiple requests to the database, but that also moves complexity into the application and is usually slower than a join performed by specialized code inside the database. In such cases, using a document model can lead to significantly more complex appli‐ cation code and worse performance

**Schema flexibility in the document model**

Document databases are sometimes called schemaless, but that’s misleading, as the code that reads the data usually assumes some kind of structure—i.e., there is an implicit schema, but it is not enforced by the database. 所谓的schemaless


### Storage and Retrieval

**storage engines**

- log-structured storage engines
- page-oriented storage engines, such as B-trees

In this book, **log** is used in the more general sense: an **append-only** sequence of records. It doesn’t have to be human-readable; it might be binary and intended only for other programs to read.

索引

An index is an additional structure that is **derived from the primary data**. Many databases allow you to add and remove indexes, and this doesn’t affect the contents of the database; it only affects the performance of queries. Maintaining additional structures incurs overhead, especially on writes.

**Hash Indexes**

keep an **in-memory hash map** where every key is mapped to a byte offset in the data file—the location at which the value can be found. 

In this kind of workload, there are a lot of writes, but there are not too many distinct keys—you have a large number of writes per key, but it’s feasible to keep all keys in memory. 适用场景是distinct key的数量不多，而且大部分操作是针对同一个key的update

局限性

- The hash table must fit in memory, so if you have a very large number of keys, you’re out of luck. 不适用于有很多key的场景
- Range queries are not efficient. 区间查询效率不高，因为只能loop所有的key

**SSTables and LSM-Trees**

SSTable(Sorted String Table): 日志中的记录是按`key`排序的，且`key`在每个文件中唯一

它有如下优势：

- Merging segments is simple and efficient. 因为每个日志文件都是按`key`排序的，可以利用`merge sort`中的算法来快速merge多个文件。
- In order to find a particular key in the file, you no longer need to keep an index of all the keys in memory. 不需要将所有的`key`索引在内存里了，可以一个`key`对应一个块(segment)，这叫sparse index(稀疏索引？)。若要找某个key，可以直接确定它大概在哪个块里，而块里的`key`都是有序的，搜寻也很快。

那么如何创建和维护这个SSTables呢？

很明显，key的写入肯定是乱序的，但我们可以先把它们写入内存，然后按序读出再写入硬盘（红黑树、AVL树等都可以做到），现在我们的算法变成：

- 当要写入一个key的时候，我们先把它写入一个平衡树（红黑树、AVL树等）中，我们一般叫这个平衡树的数据结构为`memtable`
- 当`memtable`越来越大超过某个阈值（通常为几M）时，我们把它写入一个`SSTable`文件中。直接中序遍历这棵树就能得到按序排列的key
- 当要查找某个key时，我们先在`memtable`中找，若没有，再从`SSTable`文件中找

这个方案有个问题，当数据库突然挂掉时，在`memtable`但还没有写入`SSTable`文件中的这些数据会丢。一个解决办法是：在写入`memtable`的同时，也写一份到append-only日志里，当然是乱序的，但不要紧，我们只是用它来恢复`memtable`的。

LSM-Trees： 是 Log-Structured Merge-Tree 的缩写。Storage engines that are based on this principle of merging and compacting sorted files are often called **LSM storage engines**. 感觉跟`SSTable`是同一种东西的不同叫法，或者说，`LSM-Trees`是更宽泛的叫法。

Lucene（ES用到的索引引擎）也是用的类似的数据结构来实现的全文搜索。 This is implemented with a key-value structure where the key is a word (a term) and the value is the list of IDs of all the documents that contain the word (the postings list).


性能如何呢？

Even though there are many subtleties, the basic idea of LSM-trees—keeping a cascade of SSTables that are merged in the background—is **simple and effective**. Even when the dataset is much bigger than the available memory it continues to work well. Since data is stored in sorted order, you can **efficiently perform range queries** (scanning all keys above some minimum and up to some maximum), and because the disk writes are sequential the LSM-tree can **support remarkably high write throughput.** 亮点是：支持区间查询、写入时性能高。

**B-Trees**

与 log-structured indexes 的区别：

- The log-structured indexes we saw earlier break the database down into **variable-size segments**, typically several megabytes or more in size, and always write a segment **sequentially**.
- B-trees break the database down into **fixed-size blocks or pages**, traditionally 4 KB in size (sometimes bigger), and read or write one page at a time. This design corresponds more closely to the underlying hardware, as disks are also arranged in fixed-size blocks.
- The basic underlying write operation of a B-tree is to overwrite a page on disk with new data. This is in stark contrast to log-structured indexes such as LSM-trees, which only append to files (and eventually delete obsolete files) but never modify files in place. B-tree 会 in-place update，而 LSM-tree只会 append（这是函数式编程的风格）

![](/images/post/b-tree.jpg) 

The number of references to child pages in one page of the B-tree is called **the branching factor**. For example, in Figure 3-6 the branching factor is six. In practice, the branching factor depends on the amount of space required to store the page references and the range boundaries, but typically it is **several hundred.** 其实就是这个棵树的某个节点有多少个分叉

如何update某个key / val呢？

从 root 开始，类似二分查找，直到找到包含这个key的page，然后更新即可，注意更新是以`page`为单位的

如何insert呢？

同样也是先查找，找到需要插入的节点（page）后，插入即可，但这里有一个问题，当要插入的page已经太大了时，需要把它split成两个。

经验：

a B-tree with n keys always has a depth of O(log n). Most databases can fit into a B-tree that is **three or four levels** deep, so you don’t need to follow many page references to find the page you are looking for. (**A four-level tree of 4 KB pages with a branching factor of 500 can store up to 256 TB**.)

当数据库突然挂掉时，B-tree 仍然也会丢数据，解决方案仍然也是万金油：append-only log，不过在这里是叫：write-ahead log（WAL, also known as a redo log）This is an append-only file to which every B-tree modification must be written before it can be applied to the pages of the tree itself. When the database comes back up after a crash, this log is used to restore the B-tree back to a consistent state.

**Comparing B-Trees and LSM-Trees**

As a rule of thumb, **LSM-trees are typically faster for writes, whereas B-trees are thought to be faster for reads** [23]. Reads are typically slower on LSM-trees because they have to check several different data structures and SSTables at different stages of compaction.

LSM tree 的优势：

- LSM tree 的写放大（write amplification）相对于b-tree要小
- LSM tree 更容易被压缩，且不容易出现文件碎片，磁盘利用率更高

LSM tree 的劣势

- compaction process 会可能会影响到读和写，导致读写性能不稳定，而 B-tree 是很稳定的。
- 在极端情况下，compaction process 的速度会赶不上记录被写入的速度，导致文件系统被写爆（需要一个监控）
- LSM tree 会出现一个 key 存在于多个 segments 文件的情况，而 b-tree 能保证一个key只在一个 index 文件中，因而也更容易锁住它，更容易实现事务。

**Other Indexing Structures**

> Storing values within the index

The key in an index is the thing that queries search for, but the value can be one of two things: **it could be the actual row (document, vertex) in question, or it could be a reference to the row stored elsewhere.** 索引里可以直接放真实的记录，也可以放记录的指针

In the latter case, the place where rows are stored is known as a **heap file**, and it stores data in no particular order. The heap file approach is common because it avoids duplicating data when multiple secondary indexes are present: each index just references a location in the heap file, and the actual data is kept in one place. 这种方式可以节省空间，比如多个二级索引可以复用一个记录指针，真实记录只在 heap 中存一份即可

In some situations, the extra hop from the index to the heap file is too much of a per‐ formance penalty for reads, so it can be desirable to **store the indexed row directly within an index**. This is known as a **clustered index**. 聚簇索引?

A compromise between a clustered index (storing all row data within the index) and a nonclustered index (storing only references to the data within the index) is known as a **covering index or index with included columns**, which stores some of a table’s columns within the index [33]. This allows some queries to be answered by using the index alone. 将部分字段的value一起写在index里，若查询的就是其中的字段则可以直接返回，very smart!

> Multi-column indexes

The most common type of multi-column index is called a concatenated index, which simply combines several fields into one key by appending one column to another (the index definition specifies in which order the fields are concatenated)

注意这跟上面说的 covering index 不太一样，这应该是我们平常理解的`联合索引`了

> Keeping everything in memory

in-memory databases: Redis / Memcached

Counterintuitively, the performance advantage of in-memory databases is not due to the fact that they don’t need to read from disk. Even a disk-based storage engine may never need to read from disk if you have enough memory, because the operating system caches recently used disk blocks in memory anyway. Rather, they can be faster because they can avoid the overheads of encoding in-memory data structures in a form that can be written to disk. 确实挺反直觉的，原来内存数据库快是快在了“避免了从内存数据结构编码成字节”。一般直觉会认为，传统依赖硬盘的数据库的慢是因为它们要读盘、写盘，但其实由于操作系统的`虚拟内存`机制，大部分的查询都只在内存就能满足了，无需走硬盘。

**online transaction processing (OLTP) vs online analytic processing (OLAP)**

前者是在线交易型需求，后者是分析分析型需求

In the early days of business data processing, a write to the database typically corre‐ sponded to a commercial transaction taking place: making a sale, placing an order with a supplier, paying an employee’s salary, etc. As databases expanded into areas that didn’t involve money changing hands, the term transaction nevertheless stuck, **referring to a group of reads and writes that form a logical unit**. `transaction` 一词的由来以及意思，注意跟数据库里的`事务`区别

OLTP的使用场景如下：

An application typically looks up a small number of records by some key, using an index. Records are inserted or updated based on the user’s input. Because these applications are interactive, the access pattern became known as online transaction processing (OLTP).

OLAP的使用场景如下：

However, databases also started being increasingly used for data analytics, which has very different access patterns. Usually an analytic query needs to **scan over a huge number of records, only reading a few columns per record**, and calculates aggregate statistics (such as count, sum, or average) rather than returning the raw data to the user. 

![](/images/post/oltp-olap.jpg)

These OLTP systems are usually expected to be **highly available** and to process transactions with **low latency**, since they are often critical to the operation of the business. Database administrators therefore closely guard their OLTP databases. They are usually reluctant to let business analysts run ad hoc analytic queries on an OLTP database, since those queries are often expensive, scanning large parts of the dataset, which can harm the performance of concurrently executing transactions. 分析型查询不要在业务库上做

**Data Warehousing**

数据仓库？

A data warehouse, by contrast, is a separate database that analysts can query to their hearts’ content, without affecting OLTP operations [48]. **The data warehouse contains a read-only copy of the data in all the various OLTP systems in the company**. Data is extracted from OLTP databases (using either a periodic data dump or a continuous stream of updates), transformed into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse. This process of getting data into the warehouse is known as Extract–Transform–Load (ETL) 

A big advantage of using a separate data warehouse, rather than querying OLTP sys‐ tems directly for analytics, is that the data warehouse can be optimized for analytic access patterns. 把 olap 数据库独立出来的一个好处是，可以针对 olap 类查询做优化

> Schemas for Analytics

数据仓库中主要用到的数据模型是：star schema(also known as dimensional modeling)，翻译成`星型结构`比较合适，里面主要分为两种表：fact table 和 dimension tables。感觉也没什么特别，就是一张主表，关联了多个子表。

The name “star schema” comes from the fact that when the table relationships are visualized, the fact table is in the middle, surrounded by its dimension tables; the connections to these tables are like the rays of a star. `star schema`名字的由来

In a typical data warehouse, tables are often very wide: fact tables often have over 100 columns, sometimes several hundred. 数据仓库中的表的字段数量通常很大，超过100，甚至几百个

**Column-Oriented Storage**

列式存储，为解决大数据查询而提出的优化方案

分析 olap 的查询场景：

- 通常只查询几个字段，一般不会`select *`

In most OLTP databases, storage is laid out in a row-oriented fashion: all the values from one row of a table are stored next to each other. 在大部分 oltp 数据库中， 数据是按行存储的，那么虽然我们只需要几个字段，但查询时需要先将这100多个字段先从硬盘中load进内存，然后再返回这几个需要的字段，就是说不管是 select 几个字段，都得把这一整行先读出来。。

那么所谓`列式存储`其实就是为了规避这个问题，反其道而行之，将数据按列存储，所有记录的某个字段存为一个文件，那么当 select 某个字段时，直接返回这个文件即可。

The column-oriented storage layout relies on each column file containing the rows in the same order. Thus, if you need to reassemble an entire row, you can take the 23rd entry from each of the individual column files and put them together to form the 23rd row of the table. 但需要注意的是，所有列文件里的记录顺序需要是一致的，要不就没法返回某条完整的记录了。

> Column Compression

列式存储还有一个好处是可以采用`列压缩`

Take a look at the sequences of values for each column in Figure 3-10: they often look **quite repetitive**, which is a good sign for compression. Depending on the data in the column, different compression techniques can be used. One technique that is particularly effective in data warehouses is **bitmap encoding** 重复意味着有压缩空间，而我们日常的数据，对于某个字段来说，它的值通常只有有限的数量。

Often, the number of distinct values in a column is small compared to the number of rows (for example, a retailer may have billions of sales transactions, but only 100,000 distinct products). **We can now take a column with n distinct values and turn it into n separate bitmaps: one bitmap for each distinct value, with one bit for each row. The bit is 1 if the row has that value, and 0 if not.**

If n is very small (for example, a country column may have approximately 200 dis‐ tinct values), those bitmaps can be stored with one bit per row. But if n is bigger, there will be a lot of zeros in most of the bitmaps (we say that they are sparse). In that case, the bitmaps can additionally be run-length encoded. 如果 n 很大，导致 bitmap 很稀疏，还可以用`run-length encoded`来进一步优化

> Writing to Column-Oriented Storage

策略跟 LSM-tree 一样，先写到内存中的balanced tree 中，再定期写入文件

> Aggregation: Data Cubes and Materialized Views

Another aspect of data warehouses that is worth mentioning briefly is materialized aggregates. As discussed earlier, data warehouse queries often involve an aggregate function, such as COUNT, SUM, AVG, MIN, or MAX in SQL. If the same aggregates are used by many different queries, it can be wasteful to crunch through the raw data every time. Why not cache some of the counts or sums that queries use most often? 先算一个临时结果

**Summary**

Disk seek time is often the bottleneck in OLTP.

Disk bandwidth (not seek time) is often the bottleneck in OLAP

### Chapter 4 - Encoding and Evolution

需求改变可能会导致数据模型改变，数据模型改变又往往需要改动代码，而通常我们的代码又不可能在瞬间部署完，两个原因：

- 服务端一般需要灰度部署
- 客户端是否更新，取决于用户的意愿

那么，就意味着我们的代码需要对新老数据结构做兼容，两个方面：

- Backward compatibility / 新代码需要兼容老结构
- Forward compatibility / 老代码需要兼容新结构

**Formats for Encoding Data**

Programs usually work with data in (at least) two different representations:

- In memory, data is kept in objects, structs, lists, arrays, hash tables, trees, and so on. These data structures are optimized for efficient access and manipulation by the CPU (typically using pointers).
- When you want to write data to a file or send it over the network, **you have to encode it as some kind of self-contained sequence of bytes** (for example, a JSON document).

Thus, we need some kind of translation between the two representations. The trans‐ lation from the in-memory representation to a byte sequence is called encoding (also known as serialization or marshalling), and the reverse is called decoding (parsing, deserialization, unmarshalling) 哈哈，编码与解码、序列化和反序列化，说的都是一个东西

> JSON, XML, and Binary Variants

JSON distinguishes strings and numbers, but it doesn’t distinguish integers and floating-point numbers, and it doesn’t specify a precision. JSON不能区分整数和浮点数，也不能指定精度

JSON and XML have good support for Unicode character strings (i.e., human- readable text), but they don’t support binary strings (sequences of bytes without a character encoding) 不支持 binary strings，解决方案是 base64

JSON 的二进制变种：MessagePack, BSON, BJSON, UBJSON, BISON, and Smile

The binary encoding is 66 bytes long, which is only a little less than the 81 bytes taken by the textual JSON encoding (with whitespace removed). All the binary encodings of JSON are similar in this regard. It’s not clear whether such a small space reduction (and perhaps a speedup in parsing) is worth the loss of human-readability. 这是 MessagePack 的情况，可以看到，二进制版本并没有比普通版本减少多少空间占用。

> Thrift and Protocol Buffers

The big difference compared to Figure 4-1 is that there are no field names (userName, favoriteNumber, interests). Instead, the encoded data contains field tags, which are numbers (1, 2, and 3).  跟上面的二进制变体的最大区别是，不需要字段名称，只要一个tag，因为可以提前定义好 schema

当新增和移除字段时，Thrift 和 Protocol Buffers 都可以做到 backward and forward compatibility

> Avro

[Apache Avro Documentation](http://avro.apache.org/docs/current/)

**Modes of Dataflow**

> Dataflow Through Databases

In a database, the process that writes to the database encodes the data, and the pro‐ cess that reads from the database decodes it. 有意思，这也算一种数据流动的方式

> Dataflow Through Services: REST and RPC

解惑几个名词：SOA、microservices architecture

This way of building applications has traditionally been called a service- oriented architecture (SOA), more recently refined and rebranded as microservices architecture. 哈哈，同样的东西换个说法又出来了

**REST vs SOAP**

REST is not a protocol, but rather a design philosophy that builds upon the principles of HTTP [34, 35]. It emphasizes simple data formats, using URLs for identifying resources and using HTTP features for cache control, authentication, and content type negotiation. REST 的特点是能借用 http 的就用

By contrast, SOAP is an XML-based protocol for making network API requests.vii Although it is most commonly used over HTTP, it aims to be independent from HTTP and avoids using most HTTP features. Instead, it comes with a sprawling and complex multitude of related standards. SOAP 的特点是能不用 http 的就不用，啥都自己搞

**The problems with remote procedure calls (RPCs)**

作者认为 RPC 从根本上就是 flawed，它的出发点是好的，想要把通过网络的访问封装成本地调用的效果，但网络请求跟本地请求有很大差异，这是绕不过去的：

- 本地调用的结果是可预测的，要么成功，要么失败。而网络请求是不可预测的，你发的请求或者对方的回复有可能会丢失，这完全不在你的控制范围之内
- 本地调用的结果是确定的，而网络调用会出现`未定状态`，比如请求超时，此时客户端可能会比较懵，不知道是否应该重试
- 本地调用同一个函数的返回耗时，不管调用多少次应该都是基本不变的，而网络请求的返回会因网络情况不同出现很大的差异
- 本地调用可以直接传指针（引用），而网络请求只能传递内存对象序列化后的字节
- 网络请求时，client 和 server 可能是用不同编程语言实现的，而不同编程语言的数据类型又不一样，两者转换时也会出问题

Custom RPC protocols with a binary encoding format can achieve better perfor‐ mance than something generic like JSON over REST. However, a RESTful API has other significant advantages: it is good for experimentation and debugging (you can simply make requests to it using a web browser or the command-line tool curl, without any code generation or software installation), it is supported by all main‐ stream programming languages and platforms, and there is a vast ecosystem of tools available (servers, caches, load balancers, proxies, firewalls, monitoring, debugging tools, testing tools, etc.) RPC 的有点是效率高点，而 REST 的巨大优势是：测试和 debug 方便，有大量的工具

**Message-Passing Dataflow**

消息的好处太多了：

- It can act as a buffer if the recipient is unavailable or overloaded, and thus improve system reliability.
- It can automatically redeliver messages to a process that has crashed, and thus prevent messages from being lost.
- It avoids the sender needing to know the IP address and port number of the recipient (which is particularly useful in a cloud deployment where virtual machines often come and go).
- It allows one message to be sent to several recipients.
- It logically decouples the sender from the recipient (the sender just publishes messages and doesn’t care who consumes them).
