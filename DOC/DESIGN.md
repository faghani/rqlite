# rqlite
You can find details on the design and implementation of rqlite from [these blog posts](http://www.philipotoole.com/tag/rqlite/).

The design and implementation of rqlite has been discussed at various Meetups:
 - The [GoSF](http://www.meetup.com/golangsf/) [April 2016](http://www.meetup.com/golangsf/events/230127735/) Meetup. You can find the slides [here](http://www.slideshare.net/PhilipOToole/rqlite-replicating-sqlite-via-raft-consensu).
 - ACM Pittsburgh [October 2018](https://www.meetup.com/ACM-Pittsburgh/events/254891163/), at the Pitt Computer Science Club. Those slides at available [here](https://docs.google.com/presentation/d/1lSNrZJUbAGD-ZsfD8B6_VPLVjq5zb7SlJMzDblq2yzU/edit#slide=id.p).

## Node design
The diagram below shows a high-level view of a rqlite v4.0 node.

                 ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐             ┌ ─ ─ ─ ─ ┐
                             Clients                            Other
                 └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘             │  Nodes  │
                                │                             ─ ─ ─ ─ ─
                                │                                 ▲
                                │                                 │
                                │                                 │
                                ▼                                 ▼
                 ┌─────────────────────────────┐ ┌─────────────────────────────────────┐
                 │           HTTP(S)           │ │                  TCP                │
                 └─────────────────────────────┘ └─────────────────────────────────────┘
                 ┌───────────────────────────────────────────────┐┌────────────────────┐
                 │             Raft (hashicorp/raft)             ││   Internode meta   │
                 └───────────────────────────────────────────────┘└────────────────────┘
                 ┌───────────────────────────────────────────────┐
                 │               matt-n/go-sqlite3               │
                 └───────────────────────────────────────────────┘
                 ┌───────────────────────────────────────────────┐
                 │                   sqlite3.c                   │
                 └───────────────────────────────────────────────┘
                 ┌───────────────────────────────────────────────┐
                 │                 RAM or disk                   │
                 └───────────────────────────────────────────────┘

## Log Compaction
rqlite automatically performs log compaction, so that disk usage due to the log remains bounded. After a configurable number of changes rqlite snapshots the SQLite database, and truncates the Raft log. This is a technical feature of the Raft consensus system, and most users of rqlite need not be concerned with this.

## rqlite and the CAP theorem
The [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem) states that it is impossible for a distributed database to provide consistency, availability, and partition tolerance simulataneously -- that, in the face of a network partition, the database can be available or consistent, but not both.

Raft is a Consistency-Partition (CP) protocol. This means that if a rqlite cluster is partitioned, only the side of the cluster that contains a majority of the nodes will be available. The other side of the cluster will not respond to writes. However the side that remains available will return consistent results, and when the partition is healed, consistent results will continue to be returned.
