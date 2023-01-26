# LSM Tree

In this article, I will introduce you: what is LSM Tree, how LSM Tree works, and why it is superior.

### LSM Tree overview

LSM Tree (Log Structured Merge) is not a *concrete* tree data structure like [B+ Tree](https://telegra.ph/btree-11-14), but an approach, a thinking, a collection of multiple algorithms, a data organization approach for data-intensive applications. It was first come up with Patrick O'Neil, Edward Cheng in 1996.

LSM Tree is a **leveled, sorted, disk-oriented** data structure. The main thought is to fully use the fact that the sequential I/O performance is much better than random I/O, to convert bulk randomly writes to a sequential write. The workload of LSM Tree is write intensive, instead of read intensive workload of B+ Tree. B+ Tree tries to organize the leaf pages together to do sequential read, while LSM Tree is trying to do more optimization on write.

![Random IO vs. Sequential IO](https://user-images.githubusercontent.com/70138429/214550043-9491893b-359f-4679-9330-3a691cc66d8b.png)

LSM Tree is designed for SSD ([Solid State Driver](https://en.wikipedia.org/wiki/Solid-state_drive)), instead of HDD  ([Hard disk drive](https://en.wikipedia.org/wiki/Hard_disk_drive)). Nowadays, with the development of technology, high capacity SSD is cheaper and cheaper. In theory, the performance difference of sequential IO and random IO of SSD is not so great a disparity like HDD. However, sequential write performance is better than write performance[1]. That is, a data structure must be good at converting random writes to sequential write , and try to reduce random read to reduce the IOPs[2].  So, LSM Tree was invented. The choice of data structure for a disk-oriented storage system depends on the disk driver. In the future, a new data structure might come because of persistent memory.

To reduce random writes, LSM Tree sacrifices some read ability. LSM Tree holds multiple versions of a K/V pair, that are stored in different sstable levels. The newer the data is, the higher level that data is in. So, a write operation is append write actually, while a read operation *might* go through all the files.

In summary, LSM Tree is a disk-oriented data structure for write intensive workload (such as NoSQL, NewSQL).

### How does LSM Tree works?

The main components of a LSM Tree implementation are: memtable (in-memory table), immutable memtable, leveled sstable (string sorted table) and WAL (Write Ahead Log).

* Memtable is a mutable in-memory data structure holding **current** sorted k-v pairs, usually skiplist.
* Immutable is a short middle state between memtable and sstable.
* Leveled sstable is sorted disk file in each level. To reduce the complexity of read operation, the storage system needs to compact sstables sometimes.
* A WAL file is for a particular memtable. When the memtable has been flushed to disk, the WAL file becomes invalid.

![LSM Tree](https://user-images.githubusercontent.com/70138429/214542477-58bf9079-60d0-4e10-bb04-d02c282f88dc.png)

As a key-value storage system, the main APIs are `Get(key)` and `Put(key,val)`:

* `Get(key)`: When client acquires to get value, first search in in-memory sorted memtable for the key, second search in immutable memtable if there is, then search in disk files. In general, each disk file contains a bloom filter and key interval  to help figure out whether there is the key, and then read sstable files from the disk. Client will get the value and return immediately, or just get an error message.
* `Put(key, val)`: When client acquires to put a key-value pair, it is somewhat easy. First, append a log record to particular WAL file (direct IO, do not go through page cache). After receives ACK from log-appending, put the key-value pair to memtable. If the memtable is full, covert it to immutable memtable and flush to disk when reaches the maximum size.

Of course, we cannot stand that the file becomes bigger and bigger. Therefore, in LevelDB implementation, we needs to compact sstables to reduce redundancy of duplicated key-value pairs. LevelDB uses leveling merge policy: the size of a sstable is fixed; the lower the level is, the more maximum file numbers are. When reaches the maximum file number, we needs to merge one of the sstable to next level. [4]

![Compaction](https://user-images.githubusercontent.com/70138429/214631016-f96e146e-bd69-464c-8073-b2d17ef0c5eb.png)

I have shown you the architecture of LSM Tree roughly. There are so many ways to optimize LSM Tree in production [5], which are beyond the scope of this article, so I might introduce some of them in latter articles.

### Why is LSM superior?

1. LSM Tree does a great trade-off on write amplification and read amplification. It is designed for write intensive application, but also has a passable read performance [7]. LSM Tree tries to do only necessary write, such WAL, dump a whole sstable, compact when necessary.
2. LSM Tree also works well in multi-cores CPU. It needs to handle with concurrent read, write, and compact flush, compaction. Concurrent read/write must increment the reference count of the being-used sstables to avoid invalid deleting. But concurrent flush or compaction acquire to manipulate a component exclusively, because it will change the metadata of a LSM Tree. On the other hand, we could do concurrent compaction if the sstables are irrelevant.
3. The optimization research of LSM Tree is hot in computer science and industry. With the development of NoSQL, LSM Tree becomes the queen of data intensive applications, while B+ Tree is still the king. When the academia comes up with a new idea, the industry applies very soon [6].

---

[1] [知乎：随机写是不是比顺序写慢？ - Qilan Yuan](https://www.zhihu.com/question/26028619/answer/32932317)

[2] [Sequential vs Random I/O on SSDs? - Austin Hemmelgarn](https://superuser.com/questions/1325962/sequential-vs-random-i-o-on-ssds)

[3] [LevelDB - Google](https://github.com/google/leveldb)

[4] [Leveled Compaction - RocksDB.org.cn](http://rocksdb.org.cn/doc/Leveled-Compaction.html)

[5] [LSM-based Storage Techniques: A Survey - Chen Luo, Michael J. Carey](https://arxiv.org/pdf/1812.07527.pdf)

[6] [WiscKey: Separating Keys from Values in SSD-conscious Storage - Lanyue Lu and so on](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)

[7] [B-Tree vs LSM-Tree - TiKV](https://tikv.org/deep-dive/key-value-engine/b-tree-vs-lsm/)