# How Pipelining Consensus Writes Speeds Up Distributed SQL Transactions

## Summary

**Background:**

- CockroachDB uses Raft consensus to propagate changes from a node to its replicas
- **Range:** collection of replica nodes
  - **Leaseholder Replica:** the main replica in charge of maintaining a source of truth and propagating write intents
- Storage uses MVCC
- **Transactions**
  - Collection SQL read/write statements
  - Any statement that manipulates data is a DML (Data Manipulation Language) statement --> write
  - Works in 3 stages:
    1. Preparation
       1. Creates transaction record (transaction status: PENDING/ABORTED/IN-FLIGHT/COMMITTED)
       2. Creates write intents for each data manipulation it intends to make
       3. Transactions that manipulate the same rows will be synchronously executed to guarantee serializability
    2. Commit
       1. Just updates the transaction record
    3. Cleanup
       1. Optimization: Replaces all provisioned write intents with committed values

**Problems:**

1. Cannot trivially order DML statements or concurrently execute them if they need data in different different replication groups
   1. Intent consensus synchronous
   2. **This results in transaction latency scaling linearly with the number of DML statements they contain.**
2. Other solutions (parallel statement execution, bufferring writes until commit) broke SQL semantics and were not easy to integrate or caused large numbers of transaction aborts for serializable transactions, respectively.
   1. Want to send write intents as soon as possible to prevent conflicts that could result in transaction aborts.

**Solutions:**

1. **Asynchronous Consensus**

   1. Need to synchronously check for SQL domain constraint validation
   2. Afterwards, **can do write immediately** and don't need to wait for them to finish consensus before returning to client because **the leaseholder replica** already knows the effect! It just needs to propagate the changes to the replicas.
   3. So, we can return early, while the leaseholder writes and commits.
   4. This means that we can make consensus asynchronous!
      1. **Why did the consensus need to be synchronous in the first place?** Didn't know the results of the SQL statements beforehand so needed all statements to complete synchronously to know final result. Now, this is no longer an issue.
   5. **Normal:**
      1. KV operation like a Put would acquire latches on the corresponding Range’s leaseholder
      2. Determine the Puts effect by evaluating against the local state of the leaseholder (i.e. creating a new write intent)
      3. Replicate this effect by proposing it through consensus
      4. Wait until consensus succeeds before finally returning to the client.
   6. Asynchronous consensus instructs KV operations to skip this last step and return immediately after proposing the change to Raft.
   7. Now, when doing intent consensus, can do intent consensus' in parallel, and if all the intent writes succeed, `COMMIT`.

2. Transactional pipelining! (mixes sync and async)

   1. Pipelines all statements that touch the same rows (makes them synchronous)
   2. Make everything else asynchronous
   3. **Edge case:** if all statements depend on one another, then will be completely synchronous

3. **Math**
   1. Goes from `O((W+1)*L_c)` to `O(L_c)`
   2. Parallel commits make it go from `O(2*L_c)` to `O(L_c)`

## Introduction

https://www.cockroachlabs.com/blog/transaction-pipelining/

- Background: https://www.cockroachlabs.com/blog/how-cockroachdb-distributes-atomic-transactions/
- Much has changed ^
- Changed from k,v store to full SQL database
  - It did so by introducing a SQL execution engine which maps SQL tables onto its distributed key-value architecture.

> > This post will focus on an extension to the CockroachDB transaction protocol called Transactional Pipelining, which was introduced in CockroachDB’s recent 2.1 release. The optimization promises to dramatically speed up distributed transactions, reducing their time complexity from O(n) to O(1), where n is the number of DML SQL statements executed in the transaction and the analysis is expressed with respect to the latency cost of distributed consensus.

## Distributed Transactions: A Recap

### Storage

- CockroachDB is built on RocksDB (Facebook K-V store).
  - Based on the https://www.cs.umb.edu/~poneil/lsmtree.pdf
  - CockroachDB is a SQL interface over a K-V DB

### Replication

- By default, every piece of data in CockroachDB is replicated across three nodes in a cluster (though this is configurable)—we refer to these as “replicas”, and each node contains many replicas.
  - Node manages replication.
  - Ensures high reliability.
- Like other modern distributed systems, CockroachDB uses the **Raft consensus protocol** to manage coordination between replicas and to achieve fault-tolerant consensus, upon which this state replication is built.
  - https://www.cockroachlabs.com/blog/consensus-made-thrive/
- Benefits of replication come at the cost of coordination latency
  - When replica proposes change, need a majority of replicas to agree to this change (i.e. quorum of 2 for cluster of 3 nodes)
  - Raft lets you do this with a single network call to each other replica in its replication group.
  - **Replication latency:** single round trip network call to the **median slowest member of the replication group** (why this?)

### Distribution

- Distributes different data across cluster, only storing a subset of total data on each node.
- CockroachDB breaks data up into `ranges` (64 MB chunks)
  - Each of these ranges operates independently and manages its own replication
- A range is made up of replicas
- **Leaseholder Replica:**
  - Range has 1 "leaseholder" replica that coordinates writes and serves reads from its local RocksDB store.
  - The replica's lease can be passed to other replicas as they see fit and has a time limit.
  - CockroachDB tries its best to make the leaseholder replica the only node to server SQL traffic (?)
    - Automated processes like [Follow the Workload](https://www.cockroachlabs.com/docs/v2.1/demo-follow-the-workload.html) try to enforce that policy
- **Problem:** Cannot trivially order operations if they need data in different replication groups
  - The protocol that's to be discussed will address this issue.

### Transactions

- ACID-compliant transactions while being distributed and scalable
- Inspired by https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36726.pdf
- Storage layer uses **MVCC** (multiversion concurrency control) to read transactions at scale
  - **MVCC:** reads do not block other reads and writes do not block other writes/reads
    - https://levelup.gitconnected.com/implementing-your-own-transactions-with-mvcc-bba11cab8e70
    - https://elliotchance.medium.com/sql-transaction-isolation-levels-explained-50d1a2f90d8f

**Process:**

1. **Preparation**

   - Creates transaction record (transaction commit status: PENDING/COMMITTED)
   - Creates write intents for each k-v data mutation it intends to make
     - Transaction telling the rest of the replicas and cluster that they should wait for this transaction to be aborted or completed before treating the targeted row(s) as the truth

2. **Commit**

   - After finishing read/write statements, executes `COMMIT` statement.
   - Just updates the status of the transaction record
   - Does CockroachDB appropriately revert the previously enacted read/write statements if the transaction is aborted?

3. **Cleanup**
   - Replace all provisioned write intents with committed values.
   - Better than having readers having to do an extra step and check transaction record status
   - Strictly for optimization.

Can read more here: https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html

## The Cost of Distributed Transactions in CockroachDB

Performance Model for the cost of distributed transactions.

Assumptions

1. Two dominant costs are storage latency and replication latency.

   - assume that Range leaseholders are collocated with the CockroachDB nodes serving SQL traffic
   - assume that the network latency between the client application issuing SQL statements and the SQL gateway node executing them is sufficiently negligible

2. Additional latency due to queuing for lock and latch acquisition is negligible.

   - With good schema design, can avoid write hotspots.

3. No chaos events (cluster operating a steady-state)

### Latency Model

```
L_txn = (W + 1) * L_c
```

Transactions pays the cost of distributed consensus once for every DML statement it executes plus once to commit.

### The “1-Phase Transaction” Fast-Path

- Has existed in CockroachDB since its inception
- prevents a transaction that performs all writes on the same Range and commits immediately from needing a transaction record at all.
- Transaction completes with only a single round of consensus

```
L_txn = 1*L_c
```

But this only works for transactions that can be committed immediately (i.e. implicit SQL transactions: transactions outside of `BEGIN ... COMMIT` block)

- Therefore, we can ignore this optimization for further analysis, as the transaction pipelining focuses on **explicit transactions.**

## Transactional Pipelining

Transaction latency scales linearly with the number of DML statements they contain.

- Effects are noticeable at scale with high replicaiton latencies.

### What is Transactional Pipelining?

Its stated goal is to avoid the linear scaling of transaction latency with respect to DML statement count. At a high level, it achieves this by performing distributed consensus for intent writes across SQL statements concurrently.

### Prior Art (a.k.a. the curse of SQL)

First tried to solve issue through **parallel statement execution**

- allowed clients to specify that they wanted statements in their SQL transaction to run in parallel
- **Problems:**
  - Clients had to change SQL statements to take advantage of problem statements
    - **Broke SQL semantics**
  - Big issue for ORMs or packages that abstract away SQL
  - If succeeded:
    - Returned fake transaction
  - If error:
    - Only returned error later in the transaction

### Buffering Writes Until Commit

1. Buffer all write operations until transaction was ready to commit
2. Then, can flush all write operations until transaction is committed
   1. Can pay the preparation write costs in parallel

**Problem:** Saw large uptick of transaction aborts for serializable transactions

- For snapshot serializable, not as important
- But for serializable, want write intents to be as early as possible in the process to prevent conflicts that could result in transaction aborts

### A Key Insight

- could separate SQL errors from operational errors
- Need to synchronously check for SQL domain constraint validation (NOT NULL, UNIQUE, etc.)
- Can do write intents immediately but don't actually need to wait for them to finish to return something to the client.
- Just need to make sure that the write succeeds before the transaction is committed.
- **Range's leasholder has all the info needed to determine the effect of a SQL write**
  - Only coordinates with peers when propagating changes (what about Raft and consensus?)
  - But don't need to worry about this bc it's not needed to replicate before knowing the effect of the write
  - Lets us push consensus out of the synchronous stage of statement execution
- Hence, we can do distributed consensus asynchronously!

### Asynchronous Consensus

Normal:

1. KV operation like a Put would acquire latches on the corresponding Range’s leaseholder
2. Determine the Puts effect by evaluating against the local state of the leaseholder (i.e. creating a new write intent)
3. Replicate this effect by proposing it through consensus
4. Wait until consensus succeeds before finally returning to the client.

Asynchronous consensus instructs KV operations to skip this last step and return immediately after proposing the change to Raft.

- Don't need to wait for consensus for each DML statement

### Proving Intent Writes

Now need to wait for in-flight consensus write to complete to "prove" the intent.

Transaction client (on SQL gateway node) asks range leaseholder if it has successfully replicated + persisted.

- If succeeded: intent proven
- If failed: return error
- If still in-flight: client waits until it finishes
  - **But, wasn't the point to not have to wait...?**
  - Need to see the code for specific details
- Instead, a transaction waits for all of its intents to be replicated through consensus in parallel, immediately before it commits.
  - Doesn't this contradict the purpose of async consensus..?
    - Nah, the transaction doesn't need to wait for it, but the COMMIT needs to wait for consensus to be valid for all ranges.
- Once all intent writes succeed, the transaction can flip the switch on its transaction record from PENDING to COMMITTED.

### Read-Your-Writes

The idea that you should be able to read what you wrote if that write transaction was committed.

With normal consensus, this is pretty trivial because each DML statement would **synchronously** result in intents that all later statements would see and treat as sources of truth.

With asynchronous consensus, this guarantee isn't as strong. For example, if a later statement in a transaction is trying to access data modified by an earlier statement before the earlier statement's consensus has resulted in an intent then need to be able to deal with.

- Statement intent propagation needs to result in consensus to be considered as truth.

How to deal with this issue?

- Transaction dependency pipeline!
  - If statements touch the same rows, make a pipeline for them.
  - So, the second statement that lies on the first statement's write waits for the first statement to finish first.
  - "Pipeline stall": ^ when statements need to wait for predecessor statements to proceed

Idea: **Mix of asynchronous and synchronous**

- If all statements depend on one another, then completely synchronous

### New Latency Model

**Idea: We can read this as saying that a transaction whose writes cross multiple Ranges pays the cost of distributed consensus twice, regardless of the reads or the writes it performs.**

```
L_txn = L_commit = 2 * L_c
```

1. Prove all intents and write to its transaction record
   1. Can be done in parallel
2. Must do operations in-order

Can actually do this in one round of distributed consensus through [Parallel Commits](parallel_commits.md).

- Parallel commits allows the two operations that occur when a transaction commits, proving provisional intent writes and flipping the switch on the transaction record, to run in parallel.
- it parallelizes the prepare and the commit phases of the 2-phase commit protocol.
