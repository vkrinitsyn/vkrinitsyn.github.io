# [Articles](https://vkrinitsyn.github.io)


## [YT Cluster](https://github.com/dbinvent/dbinvent.github.io)
**Every node accepts writes. On the Postgres you already run.**
YT Cluster: vanilla PostgreSQL plus an extension and a server process - no forked engine, no proxy in front, no managed-cloud lock-in - [dbinvent.com](https://dbinvent.com/)

Released - tested and ready: [![nightly end-to-end](https://apt.dbinvent.com/paks/nightly/badge.svg)](https://github.com/DBinvent/yt-cluster-tests/blob/main/nightly/results.yaml)
- **No data transfer** - your database becomes a cluster node where it already sits; nothing is dumped, copied or migrated
- **No vendor lock** - remove the extension and you are back to a single plain PostgreSQL, with your data in place
- **No code change** - same SQL, same drivers, same extensions; a plain `COMMIT` is a cluster commit, with no PgBouncer and no explicit completion call
- **No dedicated master** - nothing to promote, so nothing to fail over; a RAFT-based coordinator arbitrates and assigns workers, and never becomes the bottleneck
- **Writes spread across every node** - transaction boundaries are derived at runtime (virtual dynamic partitioning), so disjoint writes land on different nodes
- **Synchronization as configuration** - per-table rules decide what is redundant, what is local, and how many nodes a commit waits for: `half`, `nodes>=N`, or asynchronous
- **Resiliency in three layers** - two-phase commit on the way in; self-healing after the fact, which re-reads, votes on and repairs a divergent copy; and a row, table or node nobody can vouch for is *blocked* rather than guessed
- **Software-defined topology** (AZ, zones, tiers) - replication scope and row placement are rules, rewritten online with no redeploy or downtime
- **MPP analytics** - Apache DataFusion planned against the declared topology, ClickHouse for columnar workloads; neither needs a copy of the data
- **Asking in plain language** - `yt_llm('...')` on a local model returns PostgreSQL SQL, ClickHouse SQL or a DataFusion script, chosen by where the data lives; the model sees the catalog, never a row ([how it works](llm.md))
- **One package, not a suite** - server functions ([RPPD](/rppd#rppd---rust-python-postgres-discovery)), etcd key-value, streaming and queue ([etcd](/etcd#etcd)) and schema migration ([Schema Guard](/sg1#schema-guard)) ship with the cluster
- **C and Rust** - the extension in C, the server and tools in Rust

Measured, not asserted: **1.7x the throughput (171%)** of PostgreSQL synchronous replication on a write-only load and **1.6x (162%)** on an 80/20 read/write mix, at equal replication scope. Full synchronous two-phase commit across every node costs more - it is there for the tables that need it, not the default.

### Articles
- From 30 years of personal IT experience into 30 pillars of the solution: [30 pillars](pillars.md)
- What replication actually costs: [2PC](https://dbinvent.github.io/yt-cluster-performance.html)
- Software-defined topology, verified: [Where Rows Live](https://dbinvent.github.io/where-rows-live.html)
- Linear scalability possible: [CAP](/cap.md)
- DNS for service discovery: [DNS](dns.md)
- Questions in plain language, answered by PostgreSQL, DataFusion or ClickHouse: [LLM](llm.md)
- Resilience and membership under change, on three linked clusters: [RMC](https://github.com/DBinvent/yt-cluster-tests/tree/main/rmc) ([results](https://github.com/DBinvent/yt-cluster-tests/blob/main/rmc/results.md))

### Run it
Three writable nodes, on your laptop, in one command - the package comes from the public repository:
```
git clone https://github.com/DBinvent/yt-cluster-tests.git
cd yt-cluster-tests
APT_HOST=apt.dbinvent.com APT_HOST_IP= ./docker.sh

# then, inside the container - a three-node cluster:
cargo make --env PG_VER=14 --env CNT=3
```
Point `psql` at any node, write on two at once, and watch the rows appear on the third.


https://dbinvent.com/cluster/
