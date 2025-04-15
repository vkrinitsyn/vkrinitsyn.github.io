## ETCD
Etcd server PoC for messaging queue [This](https://github.com/vkrinitsyn/etcd)

## Features
- etcd API v3 compatible client using protobuf to leverage existing ecosystem
- priority is a [queue](https://github.com/vkrinitsyn/etcd/blob/main/queue.md#etcd-based-queue) implementation with order and delivery guarantee
- no message storage, cluster election, as use another cluster implementation
- ability to build into another rust application as a component, see [rppd](https://github.com/vkrinitsyn/rppd?tab=readme-ov-file#rppd---rust-python-postgres-discovery)

