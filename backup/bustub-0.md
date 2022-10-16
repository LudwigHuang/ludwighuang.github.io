# Bustub Review: 0. Disk Manager

> [CMU 15-445](https://15445.courses.cs.cmu.edu/fall2021/) Project Review: Disk Manager

In this series, I will show you how to build a disk-oriented DBMS (Database Management System) roughly. This article is about *Disk Manager*, contains: Disk Manger, Storage Layout and System Catalog.

This article mainly describes: **How the DBMS represents the database in files on disk**.

```
4. Query Planning         ^    Up   (high level)
3. Operator Execution     |
2. Access Methods         |
1. Buffer Pool Manager    |
0. Disk Manager           |  Bottom  (low level) <- Today
```

## Disk Manager

Disk Manager takes care of the allocation and de-allocation of pages within a database. It performs the reading/writing of pages to/from disk, providing a **logical file layer** within the context of a DBMS. (Use file I/O provided by the Operating System)

In bustub, relationship between disk manager & database file is one-to-one map. It is very easy to figure out what `page_id` does. If higher level ask for reading/writing page, it will call its disk manager with a unique `page_id`. Then, disk manger will know where  physical address) to read/write => file + offset(page_id * PGSIZE)

## Storage Layout

**Database file layout**:
The database file is organized as a heap file, representing as *Page Directory*. A file has a header page (`page_id` = 0) to store meta-data, which contains info of table/index {name, `root_id`}.

```
# Header Page: meta-data of table/index
 -----------------------------------------------------------------
| RecordCount (4) | Entry_1 name (32) | Entry_1 root_id (4) | ... |
 -----------------------------------------------------------------
```

**Database page layout**:
A table is organized as a doubly-linked list of pages. There is no such like header page of table, everything is **unordered** in a table. The table pages are representing as slotted pages.

If we need a table named "info", call disk manager to get the first page, and invoke `GetRootId(root_id)` by "info". Then, get 1st table page by `root_id` and invoke `TableHeap` by 1st page.

<img src="https://user-images.githubusercontent.com/70138429/196034238-56a962ef-336a-4deb-8eef-18b67ab96ffe.png"/>

More specific:
```
# Slotted page format:
---------------------------------------------------------
| HEADER | ... FREE SPACE ... | ... INSERTED TUPLES ... |
---------------------------------------------------------

# Header format (size in bytes):
---------------------------------------------------------------------------
| PageId (4)| LSN (4)| PrevPageId (4)| NextPageId (4)| FreeSpacePointer(4)|
---------------------------------------------------------------------------
---------------------------------------------------------------
| TupleCount (4)| Tuple_1 offset (4) | Tuple_1 size (4) | ... |
---------------------------------------------------------------
```

**Database tuple layout**:
In bustub, we use `RID` (`page_id`+`offset`) to identify a tuple. A tuple is a collection of many attribute values. When write to a table page, we need to serialize the meaningful values to a sequence of bytes. It is natural that when read from, deserialize the bytes to attributes (interpreted by DBMS).

There multiple types of values, I will not cover them here. If you want to know more, please refer to *src/include/storage/page*. One more thing: for varchar type value, there is always 4 bytes size just before it.

```
# Tuple format: Inlined
------------------------------
| SIZE | Attr1|Attr2|Attr3...|
------------------------------

# Varchar format: Not inlined
-----------------------
| LEN | Varchar Bytes |
-----------------------
```

## System Catalog

A DBMS stores meta-data about databases in its internal *catalog*s. In bustub, system catalog is a **non-persistent catalog**, which is designed for use by executors within the DBMS execution engine. It seems like there is no way to persistent a database with a catalog, that is we can only use in-memory database or maintain catalog some where else.

The catalog contains table info & index info, i.e., the meta-data of indexes and tables. In table info, there are: name, schema, table heap pointer and oid (Object id in catalog). A schema defines a relation with multiple columns. A column specifies an attribute with type, name and length.

```
# Schema:
----------------------
| Col1 | Col2 | Col3 |
----------------------

# Column
------------------------
| Name | Type | Length |
------------------------
```

## Summary

In summary, database is organized as a file in bustub, is a collection of tables and indexes. Tables & Indexes are representing as a collection of unordered pages containing tuples. We use disk manager to fetch/push pages from/to disk, i.e., reading/writing the database on disk.
