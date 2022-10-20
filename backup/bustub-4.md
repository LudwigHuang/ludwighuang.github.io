# 4. Concurrency Control

> [CMU 15-445](https://15445.courses.cs.cmu.edu/fall2021) Project Review: Concurrency Control

In this series, I will show you how to build a disk-oriented DBMS (Database Management System) roughly. This article is about Concurrency Control, contains: Transaction, Isolation Levels, Two-Phase Locking, Deadlock Prevention.

This article mainly describes: How the DBMS seiralizes execution of multiple transactions.

```
N. Logging & Recovery  ^ Up (high level)
4. Concurrency Control |     <- Today
N. Query Planning      |
3. Operator Execution  |
2. Access Methods      |
1. Buffer Pool Manager |
0. Disk Manager        | Bottom (low level)
```

## Transaction

A transaction is the execution of a sequence of one or more operations (e.g., SQL queries) on a database to perform some higher-level function. It is the basic unit of change in DBMS.

Transaction has 4 critical properties called "ACID":
1. A (Atomic): all or nothing
2. C (Consistency): it looks correct to me
3. I (Isolation): as if alone
4. D (Duration): survive failures

In most DBMSs, "A" is guaranteed by both *Concurrency Control* and *Logging & Recovery* "C" is not as practical for one-node DBMSs. "I" is guaranteed by *Concurrency Control*. "D" is guaranteed by *Logging & Recovery*. (This series will only cover *Concurrency Control*)

## Isolation Levels

To guaranteed Atomic, there are some conflicts we need to handle:
* **Overwriting Uncommitted Data**: T1 writes `O`, then T2 writes it, but T2 commits earlier than T1, so finally `O` is of the T1 modified value.
* **Reading Uncommitted**: T1 writes `O`, then T2 reads `O`, but T1 aborts finally, T2 reads `O` of the wrong value.
* **Unrepeatable Reads**: T1 reads object `O`, then T2 modified it, T1 cannot reads `O` of the same value.
* **Phantom Reads**: T1 reads some tuples from a table, T2 inserts a new tuple, then T1 does the same operation again but gets different results.

To solve conflicts above, we need a protocol between applications and DBMSs. Applications are responsible for prevent particular query combinations in multiple transactions. The protocol is *Isolation Level*:

![Isolation Levels](https://user-images.githubusercontent.com/70138429/196857643-830e3b07-fe49-4a49-a0c9-6f83d045bcf1.png)

## Two-Phase Locking

* Lock: High-level, guarantee multiple transactions works well
* Latch: Low-level, protect internal data structures

*Lock manager* is the DBMS component for *concurrency control*. It is responsible for forcing transaction following particular scheme to get and release latches based on the isolation levels. In bustub, we use 2PL & *Wound-Wait* as the low level architecture.

2PL (*Two-Phase Locking*) is a concurrency control protocol that determines whether a txn can access an object in the database on the fly. 2PL is a good way to guarantee *conflict serializability*. Conflict serializability means that the conflicting action in transactions is the same order as serial schedule.

As the name says, 2PL has 2 two phases Growing Phase & Shrinking Phase. If release a lock in growing phase, then enters shrinking phase. In shrinking phase, application cannot grant a lock. In strong strict 2PL (SS2PL, rigorous 2PL), a transaction holds all granted locks until commit.

For different *isolation level*, 2PL does different things.
* *Serializable*: Obtain all locks first; plus index locks, plus strict 2PL.
* *Repeatable Reads*: Same as above, but no index locks.
* *Read Committed*: Same as above, but S locks are released immediately.
* *Read Uncommitted*: Same as above but allows dirty reads (no S locks).

## Deadlock Prevention

2PL may lead to deadlock, so we need a way to handle with it.

In bustub, lock manager uses deadlock prevention. When there might be a deadlock, kill one of the transactions according strategy below.

*Wound-Wait* ("Young Waits for Old"):
* If requesting txn has higher priority than holding txn (i.e., requesting_txn_id < holding_txn_id), then holding txn aborts and releases lock.
* Otherwise requesting txn waits.

### Summary

In the end of this series, we implement a *lock manager* for parallel execution of multiple transactions. The *lock manager* uses 2PL & *Wound-Wait* scheme to grant and release locks based on *isolation levels*.

So we have built a one-node disk-oriented DBMS without *log & recovery*. If add a SQL level and a shell, it is a DBMS that can manage database well (though it cannot survive failures).

