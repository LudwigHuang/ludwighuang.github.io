# B+Tree Index

In this article, I will show you how to implement a B+Tree Index to support a wide range of indexes. Mainly about: B+Tree overview, kinds of indexes, drawbacks of B+Tree. (I will not talk too much about the performance of B+Tree -- just an introduction to it.)

Before reading the blog, please read [Index](https://telegra.ph/bustub-2-10-18) to gain enough background knowledge.

### B+Tree overview

B+Tree is a self-balanced tree that enables Search, Insertion, Deletion in O(log n) time. Since B+Tree came into the world, it is less than 10 years that it was regarded as a ubiquitous technique. For now, B+Tree is the most widely used data structure to store and organize data in data-intensive applications.

Not like B-Tree, B+Tree stores all the data(K/V pairs) in its leaf nodes so that scan in order is very fast (because of sequential reads on disk). The internal nodes in B+Tree is only used for indexing the child nodes.

When searching for a key, it walks the tree from root to leaf according the key value (very similar to binary search tree). Then search in the leaf node. When inserting a K/V pair, also walks from root to find a leaf node, inserts the K/V pair into the leaf node. It might splits the overflow node recursively, from bottom to up. When deleting a K/V pair, walks from root to find a leaf node, deletes the K/V pair from the leaf node. If the number of nodes is less than ceil(N/2), then borrow a pair from its brother (redistribute) if possible or coalesce the two not so full nodes into one. If redistribute, also need to update the partition key of the two nodes in parent node. If coalesce, delete the partition key in the parent node (of course, recursively).

B+Tree is almost the best data structure in the world, not perfect surely. (For the details of implementation, please refer to [DBSC](https://db-book.com/).)

---

Why is B+Tree superior?

1. B+Tree is usually used in data-intensive application, which acquires fast, stable read & write on disk device. Most of time, B+Tree will not do splitting or merging. And because every node can store multiple K/V pairs (usually no less than 256), B+Tree is short and fat. So every time of reading/writing, the operation will only involve 3 pages.
2. B+Tree also works well in multi-cores machine. There are many concurrency control algorithms for B+Tree. One of them is *crabbing protocol*. B+Tree is tree-organized, every operation should from root to parents and finally leaf node, so it is easy to perform locking to protect the data structure. For reading, shared-lock from root to leaf, when a child node is encountered, release the parent's shared-lock. For writing, exclusive-lock from root to leaf, when safe child is encountered, release all the parent's exclusive-lock. If an operation is not "safe" to some nodes, it is easy to manipulate on it because all the un-safe parents are locked.

### Kinds of indexes

B+Tree index is used for many kinds of indexes.

* *clustering index* / *primary index*: search key also defines the sequential order of the file.<br/>Easy to support by B+Tree, just use the search key as the key of B+Tree, and the RID of tuple as the value of B+Tree.
* *non-clustering index* / *secondary index*:  search key specifies an order different from the sequential order of the file.<br/>There is no sequential scan advantage for *secondary index*, but it is also good to store it as a B+Tree index. But for the time, we will not use the RID as the value, but the *primary index* search key to avoid moving for not-involved tuples.

### Drawbacks

1. Recovery: Because of splitting/redistributing/coalescing, an operation might involve multiple pages. If an error occurs, it is hard to recover without other information. So we need extra data structure: **Write-ahead log** (WAL). (or shadow paging)
2. Write performance: When writing, it must write at least 2 times (one for WAL, one for page itself). Not only this, randomly writes on a large index might suck. Because not all leaf pages can be maintained in Mem. -- This is where LSM-Tree comes.

------

Published by [Tech Blog - Huang Blog](http://huangblog.com/).
