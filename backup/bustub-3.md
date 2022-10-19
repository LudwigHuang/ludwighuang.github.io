# 3. Operator Execution

> [CMU 15-445](https://15445.courses.cs.cmu.edu/fall2021) Project Review: Operator Execution

In this series, I will show you how to build a disk-oriented DBMS (Database Management System) roughly. This article is about Operator Execution, contains: Architecture, Sequential Scan & Limit, Modification Operators, Aggregation, Join Algorithms, Expression Evaluation.

This article mainly describes: How the DBMS executes queries.

```
N. Logging & Recovery  ^ Up (high level)
4. Concurrency Control |
N. Query Planning      |
3. Operator Execution  |     <- Today
2. Access Methods      |
1. Buffer Pool Manager |
0. Disk Manager        | Bottom (low level)
```

## Architecture

In real system, the execution of a query is somehow complex (refer to [fntdb07](https://dsf.berkeley.edu/papers/fntdb07-architecture.pdf)). But in bustub, it is easy: for now, we do not have a SQL level at all. So, every query is organized with plans/nodes explicitly. (So, we cannot do any optimization on query planning in bustub)

For example, a trivial query `SELECT col_a, col_b FROM test_1 WHERE col_a < 500` is organized as following:

```
table_info (catalog): "test_1`
predicate: col_a < 500
out_schema: [{"colA", col_a}, {"colB", col_b}]
seq_scan_plan: {out_schema, predicate, table_info}
```

Architecture of query processing in bustub
* Processing model: *volcano/iterator model*<br/>
  Each query plan operator implements a `Next()` function.
* Plan processing: *Top-to-Bottom*<br/>
  Start with root, pull data from its children.
* Access method: *Sequential Scan*<br/>
  In general, we do not want to use seq_scan. But in [*2. Access Methods*](https://telegra.ph/bustub-2-10-18), we only implement *Hash Index*, so optimization here is impossible.

For every query plan, it is a tree constructed by many operators. A node of a tree is an operator. Inside an operator, there are processing logic (`Next()` function) and a plan node.

For example, the query below is always from top-to-bottom: *projection* operator pulls data from its *join* operator; *join* operator acquires data from its child operators. At the bottom, the tuples in a table are accessed by seq_scan.

![operator tree](https://user-images.githubusercontent.com/70138429/196722644-79aece8d-a534-4f10-b22e-d4a7cb32f2b3.png)

## Sequential Scan & Limit

Access method always comes first, so here it is.

**Sequential Scan**:
* Plan Node: predicate, table
* `Next()`: Fetch a **satified** tuple once until no tuple. (Record last tuple location by an internal cursor)

Before really diving into complex operators, the easiest operator *limit* can tell us how to use child operator.

**Limit**:
* Plan Node: limit number, child operator
* `Next()`: Fetch a tuple from *child's `Next()`* until no tuple or reaching limit. (Do not need to record last tuple location, because of child's `Next()`)

## Modification Operators

**Insert**:
* Plan Node: materialized tuples or child operator (tuples to be inserted), table
* `Next()`: If insert materialized tuples, need to record last tuple location by an internal cursor. Otherwise, insert child's `Next()`. When inserted into table, operator also needs to insert [part of] tuples to indexes.

**Delete**:
* Plan Node: child operator (tuples to be deleted), table
* `Next()`: Delete the tuple by RID from child's `Next()`. When deleted, also need to delete them from indexes.

**Update**:
* Plan Node: update attributes, child operator (tuples to be updated), table
* `Next()`: Update the tuple from child's`Next()` using given function. When updated, also need to update tuples in indexes.

Note: For Delete/Update operator in transaction, we need also save the previous value of tuple. In bustub, save it in transaction object.

## Aggregation

*Aggregation* operator is a pipeline breaker, which can stall the whole query because of `Init()`.

**Aggregation**:
* Plan Node: aggregates, having clause, group by clause, child operator
* `Init()`: Build a aggregation hash table by walking through all child's `Next()`
* `Next()`: Walk through the hash table, maintaining last location by an internal cursor. Compute output tuple via given evaluate function.

## Join Algorithms

*Join* operator is also pipeline breaker.

**Nested Loop Join** (stupid join): For every tuple in table *R*, walk through every tuple in table *S* to check whether match or not. (To be less stupid, I do all stuff of *nested loop join* in `Init()`)

* Plan Node: predicates, left child, right child
* `Init()`: Do things as the algorithm says, store all results in internal vector.
* `Next()`: Use an internal iterator to walk through one by one.

**Hash Join**:
* Plan Node: predicates, left child, right child
* `Init()`: Build a hash table on join keys on one of two tables
* `Next()`: Walk through another table to find out whether the tuple matches to any tuples in hash table. (Need to maintain one more internal member for tuples that match but not emit)

## Expression Evaluation

In bustub, *expression evaluation* is so easy that we do not need to construct an expression tree.

An expression tree is organized by numbers and mathematical/logical operators. It seems like entering another field -- *compilers*. In bustub, it is not worth talking about.

![Expression Tree](https://user-images.githubusercontent.com/70138429/196724112-5148fb56-305a-4327-a616-5acee0037d17.jpg)

## Summary

In summary, a query is parsed to an operator tree. We use *volcano model* to process data pulling from child operators. There is only one access method -- *sequential scan* for fetching bottom tuples. Everything is simplified in bustub, so read the textbook after finishing project 3.

