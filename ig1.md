# DBinvent infografics

use remotion for building a video for pitch.html

1. Create a new remotion project:

## Scene 1
Default configuration of Postgres server working with a single backend-UI and serve clients webpage.
- all components already shown and the animation of a connected dots packets
- zoom in to postgres and it's split into 3, while backend/frontend are shadowed:
  - pg_bouncer as proxy process
  - Postgres base server
  - replica postgres server

## Scene 2
- added more client - duplicated bellow,
- backend becomes red and overloaded
- backend duplicated bellow as well, with clients going to new backend, but still pointed to same postgres
- backend becomes green, but postgres becomes red and overloaded
- there is no way to duplicate postgres, because it's a single entry point
- client line finally becomes red crossed and denied

## Scene 3
- Cluster become in action starting with YtCtl - Setup, control and monitor app, not required, but helpfil
- duplicate data to 2nd node, and then 3rd node, while backend and clients already there
- zoom or show each postgres node contains from: pg_bouncer, yaxaha, postgres, optional replica; zoom out
- show postgres a pure vanilla not changed geneue postgres code
- postgres nodes talk to each other

## Scene 4
- clients shrink to one spot and cluster cut and scale down to single node 
- show easy in and out with pure Postgres without data migration - no vendor cluster lock

## Scene 5
caption: maintenance with no downtime
- continue from previous scene with cluster of 3 nodes
- show shadow of first line - wrap a frame around backend1 and node1 with title Maintenance and route clients to second line i.e. backend2 and node2
- repeat same with line 2, while line 1 is back to normal and ready to take clients again. do same with line 3.

## Scene 6
feature: serverless and etcd based queue
- continue from previous scene
- extract piece of backends with lambda-python logo and move into nodes
- zoom into nodes (no magnifier)
- each node shown a postgres server, yaxaha agent and python lambda function and etcd   
- pgbouncer becomes hidden to avoid over complicated detailed view
- show connection between etcd nodes. check rppd/rppd.etcd.svg
- place triangle with etcd closer to center with broad connection pipes than yaxaha. like etcd is more realtime than regular yaxaha agent. 
but yaxaha is more data consistent because linked to db

## Scene 7
feature: az and sharding for resiliency
- continue from previous scene
- shade everything but postgres 
- morph db to table view
- show tables as all rows mirrored on each node, so all data safe and available
- show table shard as disjoints portions and each node has a shard, so data could be lost on one node, but still available on other nodes AND grow without cap
- show table as shards overlapped, so next node have same portion as well - both benefits from two previous points - resiliency and grow without cap

## Scene 8
feature: mpp and analytics - use-case as is for big data analytics
- continue from previous scene with postgres only
- show usage of apache datafusion and ballista with a cluster of 3 nodes, but each node has a datafusion and ballista process
- show a query from backend to datafusion and ballista, which will split the query into 3 parts and send to each node for processing and return results back to backend
- show performance increase in compare to spark cluster with same number of nodes, because ballista is more efficient and less overhead than spark

## Scene 9
hierarchical for infinite scalability - usecase: as-is for global backend
- continue from previous scene with postgres tables only and disjoit shards
- each node shown 2 tables: dictionary and analytics with all data mirrored - those nodes called root
- morph each nodes to a tree with root node and child nodes - those nodes called servers, 
- zoom out and morph each child node has its 3 own child nodes - those are clients
- on servers dictionary marked as readonly
- on clients nodes analytics renamed to local 
- show data mutation from root to clients in dictionary table
- show data collection and intensive changes on locals and propagated to servers  

## Scene 10
sample use-case: MMO gaming
- continue from previous scene
- zoom into outsider node 
- morph table back to postgres
- show postgres engine with yaxaha agent and lambda function and etcd
- show the postgres build-in into app and that postgres can run independently without cluster for development speedup
- show the same postgres can be part of cluster and can be converted into cluster and become a global hierarchical cluster node with no data migration and no vendor lock

