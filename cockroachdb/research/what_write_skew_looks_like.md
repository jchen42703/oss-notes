# What Write Skew Looks Like

https://www.cockroachlabs.com/blog/what-write-skew-looks-like/

This post is about gaining intuition for **Write Skew**, and, by extension, **Snapshot Isolation**. Snapshot Isolation is billed as a transaction isolation level that offers a good mix between performance and correctness, but the precise meaning of “correctness” here is often vague. In this post I want to break down and capture exactly when the thing called “write skew” can happen.

## A quick primer on transactions

The unit of execution in a database is a transaction. A transaction is a collection of work that either completes in its entirety or doesn’t run at all.

- A.k.a. a read or write in the DB

Isolation is what allows users of a database to not be concerned with concurrency and it determines the extent to which a transaction appears as if it’s running alone in the database.

Here we’re going to focus on the relationship between just two of them: SERIALIZABLE and “Anomaly SERIALIZABLE” (sometimes known as SNAPSHOT).

## Transaction Isolation Levels

**Isolation:** any transaction will not be affected by other concurrent transactions. Ensures data reliability and security.

- each transaction happens in a distinct order without any transactions occurring in tandem
- Multiple transactions can occur as long as those transactions have no possibility of impacting the other transactions occurring at the same time.

https://www.geeksforgeeks.org/transaction-isolation-levels-dbms/

**Transaction isolation levels** are a measure of the extent to which transaction isolation succeeds. In particular, transaction isolation levels are defined by the presence or absence of the following phenomena:

- **Dirty Reads:** A dirty read occurs when a transaction reads data that has not yet been committed.
  - Problem: If we read an uncommitted row and that row gets reverted, then now we have information that is considered to have never existed
- **Non Repeatable read:** Non Repeatable read occurs when a transaction reads the same row twice and gets a different value each time.
  - Transaction T1 reads data. It gets updated concurrently by Transaction T2. That means that if T1 rereads the same data, it gets different results.
- **Phantom Read:** occurs when two same queries are executed, but the rows retrieved by the two, are different
  - Transaction T1 retrieves rows based on condition. Transaction T2 generates new rows that match the search criteria. When T1 is re-executed, it gets different rows

The four isolation levels are: - READ UNCOMMITTED - Dirty read - READ COMMITTED - REPEATABLE READ - SERIALIZABLE

**Snapshot isolation is the isolation level that achieves maximum concurrency.**

- https://www.geeksforgeeks.org/what-is-snapshot-isolation/?ref=rp
- Or see rest of blog

## SERIALIZABLE

In some sense, serializability is “perfect” isolation. This is what we get when it appears as if every transaction actually is run in the database all by itself, even though that’s probably not what’s going on under the hood (my server has lots of cores, I want to make use of them).

If we were running every transaction all by itself, this would just be called “serialized”. Since it’s just equivalent to running them individually, it’s called serializable.

**The clean definition of a serializable execution is, “one which is equivalent to some serial execution,” a serial execution being one in which we just run our transactions one at a time with no interleaving of operations due to concurrency.**

- a scheme in which every transaction logically does all of its work at a single, unique timestamp.

**As an aside, it’s worth noting that “the SERIALIZABLE isolation level” and “a serializable execution” are two distinct concepts.**

## Anomaly SERIALIZABLE (but actually SNAPSHOT)

Back in the day, when ANSI defined the SERIALIZABLE isolation level, their definition, while correct, could be interpreted to not preclude a lower isolation level now called "Snapshot Isolation", and a handful of enterprising database vendors took advantage of this fact. **The most well-known example of this is that if you ask Oracle for SERIALIZABLE, what you get is actually SNAPSHOT.**

1. A transaction in Snapshot Isolation has two significant timestamps: the one at which it performs its reads, and the one at which it performs its writes.
   1. The read timestamp defines the “consistent snapshot” of the database the transaction sees.
   2. If someone commits beyond this timestamp, the transaction will not see that information.
   3. **Syntax:** T == Transaction, Tx == Transaction_x, Rx == x's read timestamp, Wx == x's write timestamp
2. Two transactions are concurrent if they have overlapping read/write internvals (R1 < W2, R2 < W1)
   1. In snapshot isolation, databases enforce isolation by aborting and retrying concurrent transactions that have overlapping write sets. If the write sets are disjoint, then we gucci!

Difference between snapshot isolation and serializable?

- **With snapshot isolation, there's a problem called "write skew"**

## Write Skew

Consider the following scenario with two transactions `P` and `Q`:

1. `P` copies the value in register `x` to `y`
2. `Q` copies the value in register `y` to `x`

When you serially execute the transaction, `x == y`. However, snapshot isolation allows for:

1. Transaction `P` reads `x`
2. Transaction `Q` reads `y`
3. Transaction `P` writes the read value to `y`
4. Transaction `Q` writes the read value to `x`

This is a valid transaction because each transaction has a consistent view of the database and have disjoint write sets.

This ends up swapping `x` and `y`, which is not possible for serializable executions.

## Who Cares?

Is a slight lack of serializability okay?

There are serious security vulnerabilities due to a lack of serializability.

Why is serializability the way to go?

- Is the only way that makes sense
- It's a lot more work to maintain many invariant states than just one

## When is Snapshot not serializable?

Not serializable if there is a cycle in the serialization graph.

However, detecting and removing cycles in a large and constantly changing graph is really expensive.

Can detect it with a more efficient algorithm:

> > Theorem 2.1 is the basis of the “Serializable Snapshot Isolation” algorithm used today in Postgres: Running in Snapshot Isolation, every transaction tracks whether it is involved in a rw-dependency on either side, and if it is on both ends of an rw dependency, it (or its successor, or its predecessor) gets aborted. This conservatively aborts some transactions which are not involved in cycles, but definitely prevents all cycles. This approach is outlined in Serializable Isolation for Snapshot Databases.

There are 5 cases:

...

See article
