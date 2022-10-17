# 1. Buffer Pool Manager

> [CMU 15-445](https://15445.courses.cs.cmu.edu/fall2021/) Project Review: Buffer Pool Manager

In this series, I will show you how to build a disk-oriented DBMS (Database Management System) roughly. This article is about Buffer Pool Manager, contains: Buffer Pool Manager, LRU Policy, Parallel Buffer Pool Manager.

This article mainly describes: **How the DBMS manages its memory and move data back-and-forth from disk?**

```
N. Logging & Recovery  ^ Up (high level)
4. Concurrency Control |
N. Query Planning      |
3. Operator Execution  |
2. Access Methods      |
1. Buffer Pool Manager |                    <- Today
0. Disk Manager        | Bottom (low level)
```

## Buffer Pool Manager

Why do we need a bpm (buffer pool manager)? Because database is usually huge and [`mmap` sucks](https://telegra.ph/mmap-internals--why-it-sucks-in-DBMS-10-08), we need a more efficient way to manage our memory and move data. That is what bpm does.

In short, a bpm maintains a buffer pool which stores multiple **page copies**. The bpm doesn't know anything about pages. The higher-level components manipulate pages by bpm.

In bustub, the bpm is very simple. There are only four main operations: FetchPage, DeletePage, UnpinPage and FlushPage. Before entering implementations, there are something we need to know about:
* `page_id`: Unique identifier for physical **page on disk**
* `frame_id`: Unique identifier for physical **page in buffer pool**<br/>
  Buffer pool will hold a fixed-size pages in memory, we use `frame_id` to manipulate the specific page.
* `page_table`: Keep track of where physical pages go in the buffer pool<br/>
  Maintain a {`page_id`, `frame_id`} map, then we can use `page_id` to find out whether it is buffered or not.
* `free_list`: Free pages in buffer pool, they are not used yet.
* meta-data: In real world, page table also maintains some meta-data of pages. In bustub,
  they are stored in page itself.<br/>`dirty_flag` shows whether the page has been modified after copying
  it from disk;<br/>`pin_cnt` maintains the reference count of the page, we cannot
  evict a pinned page if `pin_cnt` is not zero.

<img src="https://user-images.githubusercontent.com/70138429/196165664-8f359d76-2cd7-4d21-9a01-e0e9eef1b3d4.png" alt="Page Table" style="zoom: 33%;" />

Now, go to the business:
* FetchPage: fetch a page from either buffer pool or disk
* DeletePage: delete a page in buffer pool and disk (deallocate it)
* UnpinPage: when higher-level components do not need a page, tell bpm it is okay to evict it from buffer pool
* FlushPage: flush dirty page to disk, persistent the page

## LRU Policy

Buffer pool is much smaller than disk, so we need to evict page when buffer pool is full. This acquires us to propose an efficient, less wasteful replacement policy. In bustub, we use LRU Policy.

LRU is the shortcut of Least Recently Use. As the name says, we always remove the least recently use page from the buffer pool. There are many ways to implement it, such as timestamp, queue...

There are many other policies, such as LRU-K Policy, Clock Policy. Easy to implement, too.

## Parallel Buffer Pool Manager

In bustub, we use multiple bpms to optimize accessing to pages. Every bpm maintains individual buffer pool. We can not only decrease the size of each buffer pool, but also call them concurrently.

Dive into implementation, it is easy: when need a page, just modulo the `page_id` to get bpm, use bpm to do stuff. This is also called *Hashing*.

There are many possible optimizations: pre-fetching according the query plan, scan sharing to reuse the data.

## Summary

In summary, buffer pool manager is responsible for managing memory and moving data back-and-forth on disk. Buffer pool manager provides an efficient guarantee, so that the higher-level components use bpm to fetch/flush pages.

