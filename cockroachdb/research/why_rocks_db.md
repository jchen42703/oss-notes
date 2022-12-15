# Why we built CockroachDB on top of RocksDB

## Introduction

If, on a final exam at a database class, you asked students whether to build a database on a log-structured merge tree (LSM) or a BTree-based storage engine, 90% of your students would probably respond that the decision hinges on your workload. “LSMs are for write-heavy workloads and BTrees are for read-heavy workloads”, the conscientious ones would write. If you surveyed most NewSQL (or distributed SQL) databases today, most of them are built on top of an LSM, namely, RocksDB. **You might thus conclude that this is because modern applications have shifted to more write-heavy workloads. You would be incorrect.**

In our case, the compelling drivers were its **rich feature set** which turns out to be necessary for a complex product like a distributed database. At Cockroach Labs we use RocksDB as our storage engine and **depend on a lot of features that are not available in other storage engines**, regardless of their underlying data structure, be it LSM or BTree based.

## What is a storage engine?

A storage engine’s job is to write things to disk on a single node.

- Storage engine's job to focus on **A**tomicity and **D**urability
- The higher layers of the database handle **I**solation and **C**onsistency
- Performance is also very important for a storage engine

Postgres doesn't have a defined storage engine, but is a giant DB monolith.

## What is RocksDB?

- Single-node K-V storage engine
- Based on log-structured merge trees (LSMs)
- Keys and values are stored as sorted strings in files called **SSTables**.
- Explains how SSTables and compaction in RocksDB works (similar to B-tree?)
  - Went over my head
- Above the on-disk levels is an **in-memory memtable**
  - Sorted in-memory data structure (concurrent skip-list in CockroachDB, but RocksDB has many options)
    - Makes reads cheap (like a cache)
  - Persisted as an unsorted Write-Ahead-Log (WAL)
    - Need this bc as memtable increases in size, need to flush contents to disk

## Translating higher level SQL operations into K and V operations

- These SQL operations are translated down into key and value operations over a single logical keyspace
- RocksDB supports many critical options in their K-V interface, such as:
  - Scanning over a range `[start, end)`
  - Deleting a range
  - Bulk ingest a set of keys and values
- RocksDB makes all of these operations performant, so it lets CockroachDB build a SQL interface over their K-V interface

## Fast scans

Scans are much more common than you might think.

**The use of MVCC causes additional complications:**

- Cockroach stores multiple values of a given key + the timestamp at which each key was written.
- Transactions happen at a particular timestamp, and are guaranteed to read the latest value as of that timestamp.
- Thus, what may appear to be a `GET` operation at the database level (e.g. `SELECT \* from tablename WHERE primarykeycol = 123` translates into a `GET` for the value of the key that stores that row) turns into a `SCAN` at the storage engine level (`SCAN` for the newest value of that key). Thus each CockroachDB key is annotated with a timestamp to create the equivalent RocksDB key. And updates to a key create additional RocksDB keys.
- Many key-value storage engines have fast GET operations, but slower SCAN operations.
- RocksDB has **prefix bloom filters**, which allows the bloom filter to be constructed on a prefix of a key. This means that a scan that is over that prefix can benefit from using the bloom filter. Without this, a storage engine will have to scan every level on every logical `GET` operation, a huge performance hit. Prefix bloom filters, like RocksDB provides, are thus table stakes for an MVCC database like CockroachDB.

## RocksDB Snapshots

...

Explicit snapshots..?

## Blitzing through more RocksDB features

### SSTable ingestion

More optimal writes with how RocksDB writes?

### Custom key comparators

Lets you define a custom key comparator, such as comparing key timestamps.

### Range Deletion Tombstones

Efficiently deleting an entire range of keys is required, otherwise operations like `DROP TABLE`s or simply moving a chunk of data between nodes can block for really long periods of time.

### Backwards iteration

Other storage options do not let you backwards iterate efficiently.

i.e. `SELECT * FROM TABLE ORDER BY key DESC LIMIT 100`

### Indexed Batches

Batches are the unit of atomic write operations (containing set, delete, or delete range operations)

RocksDB also supports an “indexed batch” that allows reads from the batch.

- lets CockroachDB read and write in a transaction

...

### Encryption Support

Self-explanatory

## RocksDB at CockroachDB today

**Problem:** RocksDB written in C++, but CockroachDB written in Go. Crossing the `CGo` barrier is delicate - and involves a **70ns** overhead per call. That sounds fast (a nanosecond is a billionth of a second), but we perform a lot of RocksDB calls. Minimizing the number of CGo crossings has a measurable impact!

RocksDB includes still more features that we do not (yet?) use in CockroachDB, such as column families, FIFO compaction, backups and checkpoints, persistent caches, and transactions
