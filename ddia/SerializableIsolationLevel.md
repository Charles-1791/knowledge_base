# Serializable - The Most Stringent Isolation Level

## Introduction

`Read Committed` solves dirty reads and dirty writes by maintaining at most two versions for each entry; `Repeatable Read` avoids non-repeatable reads through `MVCC`. However, in SQL standard, neither of these two solves `phantom read`. Thus, we introduce `Serializable` isolation, which guarantees that:

> Even though transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially, without any concurrency.
>
> \- Design Data Intensive Application

## Implementation Strategy

In general, there are three approaches to achieve `Serializable Isolation`:

- Actual Serial Execution (Redis)
- Two Phase Locking 
- Serializable Snapshot Isolation (PG)

### Actual Serial Execution

The database is running in single threaded - there is no concurrent access so no locking is required. 

This gives rise to a problem: if a user starts a transaction and never commits it, the DB stays in idle state forever, while other transactions wait to get started. Most DBs handle this by disabling interactive multi-statement transactions, forcing users to combine their statements into a single execution block and submit it as a stored procedure. For instance, `Redis` supports `lua`, `VoltDB` uses `Java` or `Groovy`.

Another problem is that, single threaded DB provides limited throughput and puts high demands on single core performance. A partial solution is partitioning - divide the dataset into non-overlapping partitions, and each partition is handled by a separate transaction-processing thread. But for any cross-partition transactions, extra coordination is required, for instance, locks must be obtained following certain order, otherwise deadlock would be a common occurrence. 

### Two Phase Locking

`2PL` is a pessimistic strategy - a transaction must add `S` locks for entries it reads and X locks for those it writes. It is like turning every `select` into `select for share`. In addition, gap locks must be added to prevent other transactions from inserting new tuples into certain range, but in real DB products, manufacturers such as `MuSQL` often implement index-range locking,  "which is a simplified approximation of predicate locking".

### Serializable Snapshot Isolation

`SSI` was first described in 2008 and is the subject of a Phd student named _Michael Cahill_. `SSI` is an optimistic approach - it runs in `SI` (`Repeatable read`) and requires no locking for reads. Nevertheless, it maintains a graph of happens-after relationship among transactions and detects cycle in it. Once a cycle is found, one victim transaction is aborted.

## 2PL VS SSI

Let's look at an example: table `t1` and `t2` have the following structure and entry:

| table1 |
| :----- |
| A int  |
| 1      |

| table2 |
| ------ |
| A int  |
| 2      |

### Case 1

Suppose there are two transactions - `TX` and `TY` - trying to access these two tables, and follows the execution sequence:

0. both `TX` and `TY` begins
1. `TX` read from `table1`.
2. `TY` read from `table2`.
3. `TX` tries to update the entry in `table2` from 2 to 3. 
4. `TY` tries to update the entry in `table1` from 1 to 0.

**SSI**

If the DB is running in `SSI`: in step 3, when transaction `TX` adds a new version to the head of the chain, it notices that `TY` has read the old version, so `TX` must `happens after TY`. While in step 4, when `TY` is adding a new tuple to the chain head, it notices that `TX` has read the old version, so `TY` must `happens after TX`. 

Immediately, we found a `happens-after` loop. When the two transaction commits, DB would choose the later one to sacrifice.(postgres)

```
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

**2PL**

By comparison, let's see what happens if we use two phase locking.

Step 1 and Step 2 run smoothly and add `S` locks to these two entries. At step 3, when `TX` tries to update table2's entry, it cannot obtain an `X` lock because `TY` is currently holding an `S` lock on the entry, so it hangs. Similarly, at step 4, `TY` cannot proceed because it needs to obtain an `X` lock while `TX` is holding an `S` lock. 

Now both of the transactions hang, forming a deadlock. Luckily, most 2PL based databases, such as `MySQL`, have deadlock detection, which chooses one transaction to abort.

**Comparison**

In this case, both 2PL and SSI causes one transaction to rollback, but SSI does not apply S locks, so it is slightly better than 2PL.

### Case 2

Now let's reorder the four steps and see what happens:

0. both `TX` and `TY` begins
1. `TX` read from `table1`.
2. `TX` tries to update the entry in `table2` from 2 to 3. (not commit upon finishing)
3. `TY` read from `table2`.
4. `TY` tries to update the entry in `table1` from 1 to 0.

**SSI**

Running under `SSI`, a DB  executes step 1 and 2 normally without `happens-after` relationship added. 

At step 3, `TY` wants to read an entry in table2, when iterating the version chain, it notices the version inserted by `TX`, which should be ignored, so it just returns the old value `2`. This adds an happens-after relationship: by definition of serializable, the query results can be achieved by sequentially executing these two transactions; `TY` sees(returns) a data not been modified, so `TY` must happens before `TX`, namely, `TX happens after TY`. 

Nevertheless, at step 4, `TY` updates an entry in table 1, but it observed that the old entry was read by `TX`, so it adds a new version to the chain and create a happens after relationship: `TY happens after TX`.

Just like before, there is a cycle and when the latter to commit gets aborted (postgres).

**2PL**

DB executes step 1 and 2 normally, adding `S` lock on entry in `table1` and `X` lock on entry in `table2`. When `TY` attempts to read from `table2` in step 3, it gets blocked and can't continue. Only after `TX` commits can `TY` proceeds. So there is not transaction getting aborted.

**Comparison**

In this case, `2PL` is slightly better - `SSI` wastes CPU executing the second transaction, which is later aborted and requires retry, while `2PL` just prevents the second transaction from proceeding.

### Case 3

0. both `TX` and `TY` begins
1. `TX` tries to update the entry in `table1` from 1 to 0. (not commit upon finishing)
2. `TX` tries to update the entry in `table2` from 2 to 3. (not commit upon finishing)
3. `TY` read from `table2`.
4. `TY` read from `table1`.

Under `SSI`,  when DB is execute step 3, returns the old value 2, forcing a `TX happens after TY` relationship. At step 4, DB returns the old value 1, forcing the same happens-after relationship. As a result, `TY` acts as if it happens before `TX`, apparently serializable. In contrast, `2PL` blocks step 3 unnecessarily, forcing `TY happens after TX`.

In this case, `SSI` is apparently better than `2PL`.

### Summary

 `2PL` behaves like a first-come-first-serve system: semantically, the transaction that acquires a lock first logically `happens before` the ones waiting for it. However, adding read lock introduces extra overheads and can be an overkill for read-only transactions. Moreover, more locks means higher risk of deadlock. On the positive side, it does free transactions from retrying - it converts retrying into waiting.

`SSI` optimistically assumes transactions can be serialized without conflicts. When it is impossible, it chooses a victim to rollback, forcing a retry. This strategy achieves higher concurrency but can be a bad option if transaction retrying is expensive.

`2PL` uses blocking to prevent anomalies; `SSI` uses aborts to fix them.