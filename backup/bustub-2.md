# 2. Access Methods

> [CMU 15-445](https://15445.courses.cs.cmu.edu/fall2021/) Project Review: Access Methods

In this series, I will show you how to build a disk-oriented DBMS (Database Management System) roughly. This article is about Access Methods, contains: Index, Index Page Layout, Extendible Hash Table.

This article mainly describes: **How to support the DBMS's execution engine to read/write data from pages**

```
N. Logging & Recovery  ^ Up (high level)
4. Concurrency Control |
N. Query Planning      |
3. Operator Execution  |
2. Access Methods      |                   <- Today
1. Buffer Pool Manager |
0. Disk Manager        | Bottom (low level)
```

## Index

DBMSs need some data structures for: Internal Meta-data, Core Data Storage, Temporary Data Structures, Table Indexes. In this article, we only concern about table indexes.

There are two main types of indexes: *Hash Index*, *Ordered Index* (B-Tree). *Ordered index* is sufficient in almost everything: range scan, comparation, point-finding. *Hash index* is sufficient in point-finding, bad in others.

Table index is much smaller than the entire table, and the content must be synchronized to the table. Indexes are critical for efficient processing of queries in databases. Without indexes, every query would end up reading the whole content of tables it needs. In general, indexes will store {key,rid}s.

This article only covers *hash index*. *Hash index* has a hash table container to deal with point-finding, which maintains *directory page*s and *bucket page*s. Our *hash index* only supports point-finding and scan key and all stuff is implemented in container.

## Index Page Layout

In extendible hash table, there are two types of index page: *directory page* and *bucket page*.

**directory page**:

Record page id for each bucket. To locate a bucket, use the LSBs of hash value on a key; then, get the bucket's page id via one-to-one array.

Each directory page maintains a global depth and each bucket's local depth. When we need a tuple, first hash it on its key, then use the LSBs to locate the bucket, search the tuple on the bucket linearly.

```
# directory page:
--------------------------------------------------------------------------------------------
| LSN (4) | PageId(4) | GlobalDepth(4) | LocalDepths(512) | BucketPageIds(2048) | Free(1524)
--------------------------------------------------------------------------------------------
```

**bucket page**:

Store K-V{key, rid} pairs; Search linearly by hash value.

Each bucket page maintains an array of K-V pairs. For convenience, it also maintains bitmap for each K-V pairs.

```
# bucket page:
--------------------------------------------------------------------------
| Bitmap | KEY(1) + VALUE(1) | KEY(2) + VALUE(2) | ... | KEY(n) + VALUE(n)
--------------------------------------------------------------------------
```

## Extendible Hash Table

For hash table, there are two critical design decisions:
* *Hash Function*: Low hash collision, high speed.<br/>
  In bustub, we use third-party MurmurHash.
* *Hashing Scheme*: Handle collisions, quick search.<br/>
  In bustub, we use a dynamic hashing scheme called *extendible hash table*

There are three operations on extendible hash table:

**`GetValue`**:

1. Fetch directory page `dir_page`
2. Fetch bucket page `bkt_page` by hash(key)
3. Search linearly to find the tuple by hash(key)

Lock Scheme: RLock on bucket page; RLock on table (directory page).

**`Insert`**: Check whether the bucket is full: if not, just insert it. Otherwise, follow the steps below:

0. Fetch `bkt_page` by hash(key)
1. Before splitting, increment local depth
2. Check if hash table has to **grow directory**: if is(LD > GD), increment GD
3. Initialize a split image bucket page `img_page`
4. Re-hash the existing k/v pairs (in `bkt_page`) into `img_page` and `bkt_page`
5. Re-organize bucket page pointers in directory page: the prev half points to `bkt_page`, the next half points to `img_page`
6. Insert the k/v until success or fail

Lock Scheme: WLock on bucket page; RLock on table if no splitting, WLock otherwise.

**`Remove`**: Remove k/v, check whether the bucket is empty: if is, return. Otherwise, follow the steps below:

1. Check three primitives, determine to merge or not
2. Delete empty bucket page `bkt_page`
3. Prev `bkt_page` points to image bucket page `img_page`
4. Decrement local depth
5. Re-organize all buckets in directory page
6. Shrink if needed

Lock Scheme: WLock on bucket page; RLock on table if no merging, WLock otherwise.

For more details, refer to [extendible-hashing](https://github.com/nitish6174/extendible-hashing).

## Summary

In summary, *hash index* is responsible for fast data retrieval without having to search through every record in a database table. We implement hash table in extendible hash table scheme, using MurmurHash as the hash function.

