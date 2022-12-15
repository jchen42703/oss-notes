# To Read List

## Cockroach Labs Blogs

- [x] [What Write Skew Looks Like](https://www.cockroachlabs.com/blog/what-write-skew-looks-like/)
  - [Notes](what_write_skew_looks_like.md)
- [x] [Transactional Pipelining A.K.A Asynchronous Pipeline Consensus](https://www.cockroachlabs.com/blog/transaction-pipelining/)
  - [Notes](./transaction_pipelining.md)
- [x] [Why we built CockroachDB on top of RocksDB](https://www.cockroachlabs.com/blog/cockroachdb-on-rocksd/#what-is-rocksdb)
  - [Notes](./why_rocks_db.md)
  - Only like 80% done, requires more knowledge on MVCC and LSMs.
- [ ] [Parallel Commits](https://www.cockroachlabs.com/blog/parallel-commits/)
- [ ] https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/
  - Similar to the Cockroach Labs Write Skew One

Tbh, their whole blogs have bangers:

https://www.cockroachlabs.com/blog/page/4/#newpage

## Official Docs

- [ ] [Architecture Overview](https://www.cockroachlabs.com/docs/stable/architecture/overview.html)
  - [ ] [Transaction Layer](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html)
- [ ] [Follow the Workload](https://www.cockroachlabs.com/docs/v2.1/demo-follow-the-workload.html)

## Integration With Other Services

- Full-Text Search
  - [Trigram Indices](https://www.cockroachlabs.com/blog/use-cases-trigram-indexes/)
  - [CockroachDB CDC ElasticSearch](https://www.cockroachlabs.com/blog/cockroachdb-cdc-elasticsearch/)

## SQL

- [ ] [Upsert](https://www.cockroachlabs.com/blog/sql-upsert/)

## Data Structures & Algorithms

- Raft
  - https://www.cockroachlabs.com/blog/consensus-made-thrive/
- LSM Tree
  - https://www.cs.umb.edu/~poneil/lsmtree.pdf
- MVCC (Multiversion Concurrency Control)
  - https://levelup.gitconnected.com/implementing-your-own-transactions-with-mvcc-bba11cab8e70
    - https://elliotchance.medium.com/sql-transaction-isolation-levels-explained-50d1a2f90d8f
