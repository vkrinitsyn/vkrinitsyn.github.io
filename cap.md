# Solving CAP: How YaXaHa Cluster Navigates the Impossible Triangle

*Vladimir Krinitsyn*

---

## Abstract

The CAP theorem states that a distributed system can guarantee at most two of three properties: Consistency, Availability, and Partition Tolerance. For two decades, this theorem has been treated as a hard ceiling — a reason to accept painful trade-offs. YaXaHa Cluster challenges that framing not by breaking the theorem, but by making the trade-off programmable, situational, and transparent to the application. This article explains how, and walks through a live demonstration.

---

## 1. The Problem: Why CAP Still Hurts in 2026

Cloud-native databases have taught us to accept their trade-offs. DynamoDB gives you availability and partition tolerance but eventual consistency by default. Google Spanner gives you consistency and availability but only inside Google's private WAN. CockroachDB, TiDB, and YugabyteDB give you CP systems with impressive engineering — but at the cost of complexity, cloud lock-in, or a licensing model that punishes growth.

For organizations running on-premises or at the edge — manufacturing plants, hospitals, financial institutions, gaming backends, regional compliance environments — none of these options fit cleanly. You cannot run Spanner on-prem. You cannot afford CockroachDB's licensing at scale. You cannot accept Cassandra's eventual consistency for financial transactions.

The real problem is not the CAP theorem itself. The problem is that existing systems force you to pick a **fixed point** on the CAP triangle at design time, and live with that choice forever. YaXaHa Cluster's answer is to make consistency level a **runtime, per-transaction decision**.

---

## 2. What YaXaHa Cluster Is

**YaXaHa Cluster** is a distributed database system built on top of standard PostgreSQL. It is implemented in Rust and delivered as a PostgreSQL extension (`ytsetup`). It transforms any set of PostgreSQL instances into a peer-to-peer distributed cluster — with no dedicated master node, no centralized coordinator, and no single point of failure.

Key facts:
- Written in **Rust** for performance and memory safety
- Runs as a **PostgreSQL extension** — no forked or patched Postgres required
- **Leaderless by default**: any node can accept writes
- **Term-based consensus** with dynamic leader election per shard or transaction group
- **No centralized entry point**: clients connect to any node
- Supports PostgreSQL 13, 14, and newer

Source: [github.com/DBinvent/yaxaha](https://github.com/DBinvent/yaxaha)  
Product page: [dbinvent.com/cluster](https://www.dbinvent.com/cluster/)

---

## 3. The Architecture

### 3.1 Virtual Dynamic Partitioning (SSC Model)

YaXaHa uses a **Super-Shard-Cell (SSC)** model for data partitioning. Unlike static consistent hashing (used by most distributed databases), SSC partitioning is dynamic: shards are reorganized at runtime based on load, access patterns, and node availability. An AI-assisted partitioning layer monitors query patterns and proposes rebalancing without downtime or data migration.

This is a fundamental departure from systems like Citus or Greenplum, where shard rebalancing requires explicit operator intervention and often temporary performance degradation.

### 3.2 RAFT-Based Transaction Coordinator

Distributed transactions are coordinated using a **RAFT consensus variant** — but not in the traditional sense of a single RAFT group for the entire cluster. YaXaHa implements per-shard RAFT groups, so the transaction coordinator overhead scales horizontally with the number of shards. There is no global serialization bottleneck.

When a node is elected leader for a given term, it logs:
```
Term: 3, for 6sec., Leader
```
Non-leader nodes record the current leader's address and port, routing write requests accordingly without a centralized proxy.

### 3.3 Programmable Synchronization Rules

This is YaXaHa's most distinctive architectural feature. Rather than offering a single consistency model cluster-wide, it exposes **Programmable Synchronization Rules** — per-table or per-transaction policies that define:

- How many replicas must acknowledge a write before it commits (write quorum)
- Whether a read must be served from the leader replica or can be served from any replica (read consistency level)
- Whether a transaction should block on partition or degrade to local commit (partition behavior)

Example policy semantics:
```
-- Strong consistency: all replicas must ack
SYNCHRONIZE TABLE accounts WITH QUORUM = ALL;

-- Eventual: write proceeds when one replica acks
SYNCHRONIZE TABLE user_sessions WITH QUORUM = ONE;

-- Regional: write must be acked by at least one node per region
SYNCHRONIZE TABLE gdpr_data WITH QUORUM = REGIONAL;
```

This means a single cluster can simultaneously be:
- **CP** for financial transactions (accounts table)
- **AP** for session state (user_sessions table)
- **Region-consistent** for compliance data (gdpr_data)

The CAP trade-off is no longer a cluster property — it becomes a **data classification problem**.

### 3.4 Multi-Region and Edge Topology

The cluster supports hierarchical multi-region deployment. Nodes in the same data center form a local RAFT group. Data centers form a higher-level synchronization group. This allows:

- Low-latency reads from local replicas
- Configurable write propagation to remote regions
- Graceful degradation when inter-region links partition

This architecture directly addresses GDPR and data residency requirements: a table's synchronization rule can enforce that its quorum is satisfied only by nodes in a specific geographic region.

---

## 4. How YaXaHa Navigates CAP

| Scenario | Consistency | Availability | Partition Tolerance | YaXaHa behavior |
|---|---|---|---|---|
| Normal operation | Strong | Full | N/A | Synchronous replication, any node writable |
| Node failure (minority) | Strong | Full | Yes | Remaining nodes form quorum, no downtime |
| Network partition (minority isolated) | Strong | Partial | Yes | Majority partition continues; minority serves reads only |
| Network partition (50/50 split) | Degraded | Configurable | Yes | Per-table policy: block writes or accept local commits |
| Single-node "cluster" | Strong | Full | N/A | Standard PostgreSQL behavior, no overhead |

The crucial insight: **partition tolerance is always guaranteed** (you cannot opt out of it in a real distributed system). The programmable quorum rules let you choose, **per table, per transaction**, where on the C-A spectrum you want to land when partitions occur.

---

## 5. Competitive Landscape

| System | Consistency model | On-prem | Postgres-native | Write scale |
|---|---|---|---|---|
| **Citus** | Strong (coordinator bottleneck) | Yes | Extension | Limited by coordinator |
| **Greenplum** | Strong (MPP) | Yes | Forked PG | Good for analytics |
| **CockroachDB** | Serializable (CP) | Yes ($$) | No | Linear |
| **YugabyteDB** | Tunable (CP default) | Yes | API-compatible | Linear |
| **Spanner** | External consistency | No | No | Linear |
| **YaXaHa** | **Programmable** | Yes | Extension | Linear |

YaXaHa's key differentiators:
1. **No coordinator bottleneck**: per-shard RAFT groups, not a single RAFT cluster
2. **No data migration on rebalancing**: SSC model rebalances routing, not data
3. **True PostgreSQL extension**: no forked binary, runs alongside any PG version
4. **Programmable consistency**: the only system where CAP trade-off is per-table

---

## 6. Ecosystem: Supporting Tools

YaXaHa is part of a broader toolchain developed by Vladimir Krinitsyn and DBinvent:

- **[rppd](https://github.com/vkrinitsyn/rppd)** — Rust + Python + PostgreSQL integration layer (4 stars). Enables Python UDFs inside PostgreSQL backed by Rust performance.
- **[schema_guard](https://github.com/vkrinitsyn/schema_guard)** — YAML-driven PostgreSQL schema migration tool. Manages cluster-wide schema changes safely during rolling upgrades.
- **[shim](https://github.com/vkrinitsyn/shim)** — Sliding Window Histogram for timeframe analysis. Used for query pattern monitoring that feeds the AI-assisted partitioning.
- **[etcd](https://github.com/vkrinitsyn/etcd)** — Rust PoC of etcd v3 queue messaging. Informs the cluster coordination protocol design.
- **[marg](https://github.com/vkrinitsyn/marg)** — Meta-config from application arguments. Used for cluster node configuration management.

---

## 7. Current Status and Roadmap

**What is working today (MVP):**
- RAFT leader election and validation (complete)
- Multi-node write coordination (3-node cluster tested)
- PostgreSQL extension installation via `ytsetup`
- Integration test suite (3-node, parallel execution via `cargo make`)
- Docker-based isolated test environment

**2025–2026 Roadmap:**
- Web UI for cluster monitoring and shard visualization
- Serverless node lifecycle (auto-scale down idle nodes)
- Hierarchical clustering (clusters of clusters)
- Comprehensive benchmark suite vs CockroachDB / YugabyteDB
- ARM64 and edge device support

**Installation today (Debian/Ubuntu):**
```bash
# Add DBinvent package repository
# Install PostgreSQL + YaXaHa extension
apt install yaxaha-cluster
ytsetup  # enables the extension in PostgreSQL

# Start a 3-node cluster for testing
cargo make --env PG_VER=14 --env CNT=3
```

---

## 8. Demo Guide (1-Hour Technical Walkthrough)

This section outlines the live demonstration structure. Total time: 60 minutes.

### 8.1 Setup (pre-demo, before audience arrives)
- 3 terminals open, one per cluster node (NID=1, NID=2, NID=3)
- Docker running with `./docker.sh` for isolated environment
- psql connected to each node
- Monitoring output visible showing term/leader status

### 8.2 Segment 1: The CAP Problem (10 min)

**Talk track:** Open with the CAP triangle on a whiteboard or slide. Ask the audience: "Which two would you pick?" Walk through why each classic answer hurts.

**Demo:** Show a traditional PostgreSQL streaming replication setup. Cause a failover. Show the promotion gap — the window where the system is unavailable. Ask: "What happened to CAP here?"

### 8.3 Segment 2: YaXaHa Cluster Startup (10 min)

**Demo:**
```bash
# Terminal 1: start node 1
NID=1 ./start_node.sh

# Terminal 2: start node 2
NID=2 ./start_node.sh

# Terminal 3: start node 3
NID=3 ./start_node.sh
```

Show leader election output:
```
Term: 1, for 6sec., Leader       # on one node
Term: 1, Leader: 127.0.0.1:5433  # on the other two
```

**Talk track:** No master configured. No `pg_hba.conf` primary/standby dance. The cluster elects a leader for each term dynamically.

### 8.4 Segment 3: Writes and Consistency (15 min)

**Demo: Strong consistency write**
```sql
-- On node 1
BEGIN;
INSERT INTO accounts (id, balance) VALUES (1, 1000);
COMMIT;

-- Immediately on node 3 (different terminal)
SELECT * FROM accounts WHERE id = 1;
-- Should return 1000 — synchronous replication
```

**Demo: Eventual consistency write**
```sql
-- Configure session table with ONE quorum
SYNCHRONIZE TABLE user_sessions WITH QUORUM = ONE;

-- Write on node 1
INSERT INTO user_sessions (token, uid) VALUES ('abc', 42);

-- Read immediately on node 3 — may or may not be there yet
SELECT * FROM user_sessions WHERE token = 'abc';
```

**Talk track:** Same cluster, same PostgreSQL, two different consistency behaviors. The CAP choice is now a schema design decision, not a deployment decision.

### 8.5 Segment 4: Partition Simulation (15 min)

**Demo: Kill node 3 (minority failure)**
```bash
# Stop node 3
kill $(cat /tmp/pg_node3.pid)
```

Show that nodes 1 and 2 continue serving reads and writes. Show new leader election if node 3 was the leader:
```
Term: 2, for 6sec., Leader  # new leader elected
```

**Demo: Reconnect node 3**
```bash
NID=3 ./start_node.sh
```

Show it catching up — no manual intervention, no data migration.

**Demo: Network partition (advanced)**  
Use `iptables` or Docker network rules to split nodes 1+2 from node 3. Show the 50/50 behavior under different quorum policies.

**Talk track:** This is where CAP theorem bites traditional systems hard. With YaXaHa, you've already declared your preference per table. The cluster enforces your policy automatically.

### 8.6 Segment 5: Schema Migration with schema_guard (5 min)

**Demo:**
```bash
# Cluster-wide schema change, zero downtime
schema_guard apply --cluster --schema schema.yaml
```

Show that the schema change applies to all nodes, validates consistency across nodes, and rolls back atomically if any node rejects it.

**Talk track:** Distributed schema changes are one of the hardest operational problems. schema_guard treats the cluster's schema as a versioned artifact, not a per-node configuration.

### 8.7 Q&A and Discussion (5 min)

Suggested questions to prompt:
- How does this compare to Patroni + Pgpool?
- What happens if all nodes go down and come back up simultaneously?
- How do you handle long-running transactions during leader re-election?
- What's the performance overhead vs vanilla PostgreSQL?
- How does the AI partitioning actually work?

---

## 9. Key Takeaways

1. **CAP is not a binary choice** — it is a spectrum, and the right point on that spectrum depends on the data, not the system.
2. **YaXaHa makes CAP trade-offs programmable** — consistency level is a per-table, per-transaction policy.
3. **PostgreSQL-native** — no new query language, no new wire protocol, no new ops skill set required.
4. **On-premises first** — designed for environments where cloud-native distributed databases are not an option.
5. **Rust-powered reliability** — the coordination layer is memory-safe, high-performance, and production-grade.

---

## Resources

- Product: [dbinvent.com/cluster](https://www.dbinvent.com/cluster/)
- Source: [github.com/DBinvent/yaxaha](https://github.com/DBinvent/yaxaha)
- Author's tools: [github.com/vkrinitsyn](https://github.com/vkrinitsyn)
- Original article: [medium.com/@v.krinitsyn/solving-cap-1c8bf2081c23](https://medium.com/@v.krinitsyn/solving-cap-1c8bf2081c23)
- Contact: v.krinitsyn@gmail.com

---

*DBinvent — Making distributed databases accessible to organizations that cannot move to the cloud.*
