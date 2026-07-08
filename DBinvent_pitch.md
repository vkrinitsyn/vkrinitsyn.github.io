# DBinvent

**_Build your own data source, better than on cloud_**

*Chicago based deep tech startup since 2019*

---

## Problems, that many of IT projects faced

### Data consistency - when transactions are not about money transfer
AWS: "The second Enactor's clean-up process then deleted this older plan because it was many generations older than the plan it had just applied" - this is a transaction isolation issue which lead to global disaster.

### Performance on scale-up - when vertical is not an option
CAP theorem - you can't have either: Consistency, Availability or Partition Tolerance.

### Backup, restore and repeat - without compromise
That is possible, but will require a single entry point and additional manual action on failure, and double spending for unused resources.

### Complex solution exists, but the simple is possible
No solutions are as simple as RAID or NAS for Database which is a heart of many IT projects from just a website to distributed commuting.

---

## YaXaHa - The cluster like in cloud, but better and yours

[https://www.dbinvent.com/cluster/](https://www.dbinvent.com/cluster/) — ready to use **MVP**

### Strong consistency on write and eventual consistency on read
Split the performance to a 80% read and 20% write load. Always able to read, but no write until the data become actual.

### Virtual dynamic partitioning - the CAP theorem SSC
Dynamically defining transaction boundaries and spread writing load on different nodes if they are disjoint (patent provisioning) **with AI**.

### RAFT consensus for a Transaction Coordinator - not a single entry
The special node called TC act as arbiter to assign workers nodes. The arbiter does not need to process data.

### Flexible configuration
Programmatically defined synchronization rules allows splitting data to redundant for synchronization and ephemeral for a local use.

---

## Roadmap

### Serverless platform - almost done
Build your own platform like AWS lambda or Azure Server Function without a vendor lock.

### Web UI - in progress
Simplify cluster setup by using web UI and provide user-friendly cluster management, including schema changes and testing.

### Testing - in todo as the most priority to prove the durability with a time
Perform comprehensive consistency, stress and load testing. Determine SLA and TPS.

### Sharding and Partitioning - the architecture and analysis phase
From MPP to Global hierarchical cluster model to substitute client-server interaction and even a gaming / mobile API.
