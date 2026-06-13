# Why Every Database Cluster Node Should Run Its Own DNS Server

When you deploy a distributed PostgreSQL cluster, one of the first infrastructure dependencies you hit is service discovery. Which nodes are alive? Where should clients connect? How does a new node find its peers?

The conventional answer is an external DNS server, a service mesh, or a coordination service like ZooKeeper. But there is a simpler approach: embed a lightweight DNS server directly into every cluster node.

## The idea

Each node in our cluster already knows about every other node. The cluster protocol exchanges peer lists, health status, and IP addresses as part of its normal operation. This is the same data a DNS server needs to answer queries.

So instead of maintaining a separate DNS infrastructure, we embed a UDP DNS responder into every node process. Each node serves authoritative answers for a private zone (e.g., `pgcluster.lan`) using the peer data it already has. Critically, this is not a single DNS server -- it is a fully replicated DNS running on every node simultaneously. The DNS layer has no single point of failure because it does not exist as a separate service. It is part of each node, and any node can answer any query.

## How it works

The DNS server starts alongside the cluster node and shares the same in-memory state:

1. **Zone configuration** is read from the cluster config table (`dns_zone`, `dns_port`, `cluster_alias`), stored in PostgreSQL and replicated across nodes
2. **Record data** is built from the peer registry that the cluster protocol already maintains
3. **Labels** are generated from the cluster UUID (full, short, and hyphenated forms) and an alias
4. **IP addresses** come from three sources: the node's own configured host, peer IPs from the cluster protocol, and as a last resort, the node's outbound network interface

The refresh cycle runs on the same tick as the cluster heartbeat. When a node joins, leaves, or changes its IP, DNS records update within one heartbeat interval -- typically around one second.

```
  Node A                    Node B                    Node C
  +------------------+      +------------------+      +------------------+
  | App / PostgreSQL |      | App / PostgreSQL |      | App / PostgreSQL |
  |                  |      |                  |      |                  |
  | Cluster Protocol | <--> | Cluster Protocol | <--> | Cluster Protocol |
  |  (peer registry) |      |  (peer registry) |      |  (peer registry) |
  |                  |      |                  |      |                  |
  | DNS Server :5353 |      | DNS Server :5353 |      | DNS Server :5353 |
  +------------------+      +------------------+      +------------------+
         |                         |                         |
    local.pgcluster.lan       local.pgcluster.lan       local.pgcluster.lan
    -> A_ip, B_ip, C_ip       -> A_ip, B_ip, C_ip       -> A_ip, B_ip, C_ip
```

## What this enables

### Self-configuring clusters

A new node needs exactly one piece of information to bootstrap: a connection string to PostgreSQL. From the config table it reads the cluster UUID, discovers peers, joins the protocol, and immediately starts serving DNS. No Ansible playbook needs to update a BIND zone file. No Terraform run needs to reconfigure Route 53.

### Client-side load balancing

Every node answers DNS queries with the full list of healthy peer IPs. Standard DNS round-robin gives clients a simple but effective load distribution across the cluster. No dedicated load balancer required for read replicas.

### Service discovery without infrastructure

Applications on the same host (or network) can resolve `local.pgcluster.lan` to find all cluster nodes. This works for:
- Connection poolers (like PgBouncer) discovering backends
- Monitoring agents finding scrape targets
- Application services locating their database cluster
- Cross-cluster communication between database nodes

### High availability

Traditional DNS is itself a single point of failure -- if your DNS server goes down, discovery stops. With embedded DNS, every node is a DNS server. There is no central instance to lose.

If one node goes down, two things happen: its DNS server simply stops (no impact -- other nodes keep serving), and its IP is removed from the peer registry on the next failed heartbeat. Every surviving node's DNS immediately stops returning the dead node's address. Clients performing fresh lookups route around the failure automatically.

An application can query `localhost:5353` and always get a valid answer -- no network hop, no external dependency, no risk of the DNS layer being unavailable.

### Disaster recovery

When recovering from a site failure, restored nodes rejoin the cluster protocol and their IPs appear in DNS across all surviving nodes. The recovery process does not require manual DNS updates, configuration pushes, or coordination with an external service registry.

## Pre-connection discovery: DNS as the first step

The real power of embedded DNS emerges when applications treat it as a pre-connection discovery step rather than a static hostname lookup. Before opening a database connection, the client queries DNS, receives a list of candidates, and then makes an intelligent choice about which server to connect to.

This turns DNS from a simple name-to-IP mapping into an active discovery protocol. And because every node runs its own DNS server with the same replicated data, the client can query any node -- including `localhost` -- and get the full cluster picture.

### Option 1: CNAME-based routing -- cluster alias resolution

DNS can return a CNAME (canonical name) record that maps a well-known alias to the cluster. This gives applications a stable entry point that never changes, even as the underlying cluster topology shifts.

```
  Application                    DNS (any node)
      |                               |
      |-- CNAME? prod.pgcluster.lan ->|
      |<- CNAME: local.pgcluster.lan -|
      |                               |
      |-- A? local.pgcluster.lan ---->|
      |<- A: 10.0.1.10               -|
      |<- A: 10.0.2.20               -|
      |<- A: 10.0.3.30               -|
      |                               |
      |-- connect to 10.0.1.10:5432 ->|
```

An application configured with just `prod.pgcluster.lan` discovers the real cluster alias and all its IPs through standard DNS resolution. When you migrate from one cluster to another -- say, during a major version upgrade -- you update the CNAME, not the application config.

**Example: PostgreSQL connection string with DNS discovery**
```bash
# Application config -- never changes
DATABASE_URL="postgresql://app@prod.pgcluster.lan:5432/mydb"

# DNS resolves prod.pgcluster.lan -> CNAME -> A records
# libpq tries each IP in the returned list
```

**Example: Multi-cluster with environment aliases**
```sql
-- Each environment points to a different cluster alias
INSERT INTO yt_config(name, module, value) VALUES
  ('cluster_alias', 'C', 'staging')   -- staging.pgcluster.lan
ON CONFLICT (name) DO UPDATE SET value = EXCLUDED.value;

-- Production cluster on different nodes
INSERT INTO yt_config(name, module, value) VALUES
  ('cluster_alias', 'C', 'prod')      -- prod.pgcluster.lan
ON CONFLICT (name) DO UPDATE SET value = EXCLUDED.value;
```

### Option 2: Latency-based selection -- probe and pick the fastest

When DNS returns multiple A records, the application has a choice. Instead of blindly connecting to the first IP, it can probe each candidate and pick the one with the lowest response time.

```
  Application
      |
      |-- dig local.pgcluster.lan A -->  DNS
      |<- 10.0.1.10, 10.0.2.20, 10.0.3.30
      |
      |-- TCP SYN -> 10.0.1.10:5432    (2ms)
      |-- TCP SYN -> 10.0.2.20:5432    (15ms)  
      |-- TCP SYN -> 10.0.3.30:5432    (8ms)
      |
      |== connect to 10.0.1.10 (fastest) ==|
```

This is a T3 (test-three/try-three) connection protocol:

1. **Discover** -- query DNS for the cluster alias, receive the IP list
2. **Probe** -- send a lightweight probe (TCP connect or application-level ping) to each candidate in parallel
3. **Select** -- connect to the server with the lowest latency

**Example: Rust client with latency-based selection**
```rust
use std::net::{IpAddr, TcpStream, SocketAddr};
use std::time::{Duration, Instant};

fn discover_and_connect(alias: &str, port: u16) -> Option<TcpStream> {
    // Step 1: Discover -- resolve all IPs from cluster DNS
    let ips = dns_lookup::lookup_host(alias).ok()?;

    // Step 2: Probe -- measure TCP connect latency to each
    let mut candidates: Vec<(IpAddr, Duration)> = ips.iter().filter_map(|ip| {
        let addr = SocketAddr::new(*ip, port);
        let start = Instant::now();
        match TcpStream::connect_timeout(&addr, Duration::from_secs(2)) {
            Ok(stream) => {
                let latency = start.elapsed();
                drop(stream);
                Some((*ip, latency))
            }
            Err(_) => None, // node unreachable, skip
        }
    }).collect();

    // Step 3: Select -- pick the fastest
    candidates.sort_by_key(|(_, latency)| *latency);
    candidates.first().and_then(|(ip, _)| {
        TcpStream::connect(SocketAddr::new(*ip, port)).ok()
    })
}

// Usage
let conn = discover_and_connect("local.pgcluster.lan", 5432);
```

**Example: Python client with fastest-server selection**
```python
import socket
import time
from concurrent.futures import ThreadPoolExecutor

def probe(ip, port=5432, timeout=2.0):
    """Measure TCP connect time to a server."""
    start = time.monotonic()
    try:
        sock = socket.create_connection((ip, port), timeout=timeout)
        latency = time.monotonic() - start
        sock.close()
        return (ip, latency)
    except OSError:
        return None

def discover_best_server(alias="local.pgcluster.lan", port=5432):
    # Step 1: Discover
    ips = socket.getaddrinfo(alias, port, socket.AF_INET)
    unique_ips = list({addr[4][0] for addr in ips})

    # Step 2: Probe in parallel
    with ThreadPoolExecutor(max_workers=len(unique_ips)) as pool:
        results = [r for r in pool.map(lambda ip: probe(ip, port), unique_ips) if r]

    # Step 3: Select fastest
    if not results:
        return None
    results.sort(key=lambda x: x[1])
    best_ip, best_latency = results[0]
    print(f"Selected {best_ip} ({best_latency*1000:.1f}ms) from {len(unique_ips)} candidates")
    return best_ip

# Usage: build connection string with best server
best = discover_best_server()
conn_str = f"postgresql://app@{best}:5432/mydb"
```

### Option 3: Combined -- CNAME for routing, A records for selection

The two options work together. Use CNAME for logical routing (which cluster?) and the multi-IP A record response for physical selection (which server in that cluster?):

```
  Application
      |
      |  1. Resolve logical name
      |-- CNAME? myapp-db.pgcluster.lan ----> DNS
      |<- CNAME: prod.pgcluster.lan
      |
      |  2. Resolve cluster to IPs
      |-- A? prod.pgcluster.lan ------------> DNS
      |<- A: 10.0.1.10, 10.0.2.20, 10.0.3.30
      |
      |  3. T3 probe: parallel TCP connect
      |   10.0.1.10  -> 2ms   (winner)
      |   10.0.2.20  -> 15ms  (cross-AZ)
      |   10.0.3.30  -> 8ms
      |
      |  4. Connect to fastest
      |== postgresql://10.0.1.10:5432/mydb ==|
```

This pattern gives you:
- **Stable application config** -- the logical name (`myapp-db.pgcluster.lan`) never changes
- **Operational flexibility** -- re-point the CNAME during migrations, failovers, or blue/green deployments
- **Optimal performance** -- the T3 probe finds the nearest server, naturally preferring same-AZ nodes
- **Automatic failure handling** -- dead nodes fail the probe and are skipped, no TTL wait needed

### When to use which

| Scenario | Approach | Why |
|---|---|---|
| Single-region, low latency | DNS round-robin | All servers are close; probing adds no value |
| Multi-AZ deployment | T3 probe | Same-AZ server wins on latency automatically |
| Blue/green cluster migration | CNAME | Redirect traffic by changing one DNS record |
| Read replica selection | T3 probe | Pick the least loaded replica by response time |
| DR failover | CNAME + T3 | CNAME switches to DR cluster; T3 finds best node there |

## Implementation choices

A few design decisions are worth noting:

**Private zone suffix.** We use `.lan` rather than `.local`. The `.local` TLD is reserved for Multicast DNS (RFC 6762), and both systemd-resolved on Linux and Bonjour on macOS intercept queries for it. Using `.local` would silently break resolution on most modern operating systems. The `.internal` and `.lan` suffixes are safe alternatives.

**UDP only.** DNS over UDP keeps the server trivially simple. For the small record sets a database cluster produces (tens of IPs, not thousands), responses fit comfortably within a single UDP packet.

**Loopback filtering.** Nodes configured with `localhost` as their bind address must not advertise `127.0.0.1` in DNS -- that address is meaningless to remote clients. We filter loopback addresses and fall back to detecting the node's outbound IP via a non-sending UDP socket probe.

**Shared state, zero coordination.** The DNS server reads from the same peer registry that the cluster protocol writes to. There is no separate replication channel for DNS data. Every node has the same view of the cluster, so every node's DNS gives the same answers. The consistency guarantees are exactly those of the cluster protocol itself -- and the DNS layer adds no coordination overhead because it is a read-only consumer of data that already exists.

## The broader point

Infrastructure services like DNS feel heavyweight because we are used to deploying them as standalone systems. But when a process already maintains the data that a service would serve, embedding that service is often simpler and more reliable than operating it externally.

A database cluster node that knows its peers can serve DNS. A message broker that tracks consumer groups can serve service discovery. The data is already there -- the question is whether you expose it.

For our PostgreSQL cluster, embedding DNS eliminated an external dependency, simplified deployment, improved failure detection latency, and gave us self-configuring service discovery, and was under 250 lines of Rust using the [hickory-dns](https://github.com/hickory-dns/hickory-dns) library.

---

*Part of the [YT PostgreSQL Extension](https://github.com/vkrinitsyn) cluster toolkit, which also includes embedded [rppd](https://github.com/vkrinitsyn/rppd) (Replicated Persistent Priority Deque) and [etcd](https://github.com/vkrinitsyn/etcd) (Rust implementation) services for distributed coordination.*
