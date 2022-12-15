# Parallel Commits

**Background Info:** https://www.cockroachlabs.com/blog/transaction-pipelining/

**Motivation:** Distributed transactions are key to CockroachDB, so want to make them faster

- Old atomic commit protocol too slow and rigid when running a collection of individually replicated consensus groups (`ranges`)
  - A.k.a when having a bunch of replicated DBs that should have some sort of consensus on data integrity
- New atomic commit protocol: parallel commits
  - Halves latency of distributed transactions by performing all consensus roundtrips concurrently
