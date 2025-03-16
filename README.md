# [Articles](https://vkrinitsyn.github.io)

## [Schema guard](/sg1/) 
Declarative and Imperative (Flyway inspired) DB schema management.   
This Rust based opensource project is a portion of SchemaGuard complete commercial solution. 
- Designed for embed schema into Rust code.
- Support flexible schema updates if required
- Does not require history table


## [RPPD](/rppd/) 
Rust-Python-Postgres-Discovery.
Serverless platform for run Python code with Postgres:
- Postgres extension with a trigger to notify backend service about a insert or update or delete event
- Rust based opensource project with languages support (i18n)


## [YaXaHa](/yt/)
MVP of Postgres based cluster governed by RAFT algorithm:
- XA transaction based data consistency guqranee
- HA (High-availability) - no dedicated single master as entry point  
- uses standard (vanilla) PostgreSQL server with extention AND optional patched PGbouncer


## [ETCD](/etcd/)
Etcd server PoC for messaging queue   
- etcd API v3 compatible client using protobuf
- Lightweight [queue](https://github.com/vkrinitsyn/etcd/blob/main/queue.md) with order and delivery guarantee
- Rust based opensource project


## Tools

### [Marg](https://github.com/vkrinitsyn/marg) - MetaArgs for console rust apps
Fewer application configuration to connection to database and store everything else in DB.
- Available as cargo dependency in Rust:
  ``` marg = "0.3.0" ```

### [Shim](https://github.com/vkrinitsyn/shim/) - Calculate sliding average
Percentile and bucket size configuration support for histogram calculation.
