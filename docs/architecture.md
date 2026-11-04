# Architecture – PostgreSQL High Availability Lab

## 1. Overview

This lab implements a three-node PostgreSQL high-availability cluster using Patroni and etcd.

The environment runs on three Ubuntu Server virtual machines hosted by VMware. Each VM runs:

- etcd – distributed configuration store and coordination.
- Patroni – PostgreSQL high-availability manager.
- PostgreSQL – relational database engine.

The cluster uses PostgreSQL streaming replication to maintain copies of database data across the nodes.

At any given time, one node acts as the primary (leader), while the other two operate as replicas.

## 2. Infrastructure topology

| Node | Hostname | IP address | Services |
|---|---|---|---|
| Node 1 | patroni1 | 192.168.116.129 | etcd, Patroni, PostgreSQL |
| Node 2 | patroni2 | 192.168.116.132 | etcd, Patroni, PostgreSQL |
| Node 3 | patroni3 | 192.168.116.133 | etcd, Patroni, PostgreSQL |

**Infrastructure details**

- Hypervisor: VMware
- Network: VMware NAT
- Operating system: Ubuntu Server 26.04.1 LTS
- PostgreSQL: 18.6
- Patroni: 4.1.0
- etcd: 3.5.16

## 3. Logical architecture

```text
                   PostgreSQL HA Cluster
                       patroni-lab
                            |
                 +----------------------+
                 |     etcd cluster     |
                 |                      |
                 |  patroni1  patroni2  |
                 |          patroni3    |
                 +----------------------+
                            |
                     Patroni DCS
                  Leader election/state
                            |
           +----------------+----------------+
           |                |                |
      +---------+      +---------+      +---------+
      | patroni1|      | patroni2|      | patroni3|
      +---------+      +---------+      +---------+
      | Patroni |      | Patroni |      | Patroni |
      | Postgres|      | Postgres|      | Postgres|
      +---------+      +---------+      +---------+
           |                ^                ^
           |                |                |
           +------ WAL streaming replication+
```

The diagram represents the logical cluster. The primary role can move between nodes following a switchover or failover.

## 4. Component responsibilities

### 4.1 etcd – distributed coordination

etcd is a distributed key-value store used by Patroni as its Distributed Configuration Store (DCS).

Responsibilities:

- Store PostgreSQL cluster state.
- Coordinate leader election.
- Maintain cluster membership information.
- Provide distributed locking through leases.
- Help prevent multiple nodes from independently assuming leadership.

The lab uses three etcd members.

A majority of members (two out of three) is required to maintain quorum.

### 4.2 Patroni – high availability management

Patroni manages the PostgreSQL instances on each VM.

Responsibilities:

- Bootstrap the initial PostgreSQL primary.
- Configure and manage replica instances.
- Monitor PostgreSQL health.
- Coordinate leadership through etcd.
- Promote an eligible replica when the primary becomes unavailable.
- Reconfigure replicas to follow the current primary.

Patroni communicates with the local PostgreSQL instance and the shared etcd cluster.

### 4.3 PostgreSQL – database engine

PostgreSQL stores and manages the application data.

The cluster uses streaming replication:

1. The primary generates Write-Ahead Log (WAL) records.
2. Replicas receive WAL records from the primary.
3. Replicas replay the records to keep their database copies updated.

The lab uses asynchronous streaming replication.

## 5. Network communication

The following ports are used by the lab:

| Port | Protocol | Purpose |
|---|---|---|
| 2379 | TCP | etcd client communication |
| 2380 | TCP | etcd peer communication |
| 5432 | TCP | PostgreSQL connections and replication |
| 8008 | TCP | Patroni REST API |

All three VMs communicate over the VMware NAT network.

The Patroni REST API is configured to listen on port 8008, and PostgreSQL listens on port 5432.

**Security note:** These ports are exposed within the lab network. A production deployment should use restrictive firewall rules, authentication, and TLS where appropriate.

## 6. Cluster lifecycle

### Initial bootstrap

- etcd starts on all three nodes.
- Patroni starts on patroni1.
- patroni1 initializes the PostgreSQL cluster.
- patroni2 and patroni3 join as replicas.

### Normal operation

- patroni1 is the primary.
- patroni2 and patroni3 are replicas.
- Replicas continuously stream and replay WAL records.
- Patroni monitors cluster health and leadership.

### Planned switchover

A planned switchover transfers leadership from the current primary to a selected replica.

In this lab:

- Original leader: patroni1
- Selected candidate: patroni2
- New leader: patroni2

### Automatic failover

When the current primary becomes unavailable, Patroni uses the DCS to coordinate the election of an eligible replica.

In this lab:

- patroni2 was the current primary.
- Its Patroni service was stopped to simulate failure.
- patroni1 was automatically promoted.
- patroni3 continued as a replica.
- patroni2 was subsequently restarted and rejoined as a replica.

## 7. Final verified cluster state

After all tests:

| Node | Role | State | Timeline |
|---|---|---|---|
| patroni1 | Leader | Running | 3 |
| patroni2 | Replica | Streaming | 3 |
| patroni3 | Replica | Streaming | 3 |

Both replicas reported zero replication lag in the final `patronictl list` output.

## 8. Important design considerations

This lab demonstrates core high-availability mechanisms, but it is not a production-ready architecture.

Considerations for production include:

- Dedicated or appropriately isolated failure domains.
- Secure etcd and PostgreSQL communication.
- Database backup and point-in-time recovery.
- Monitoring and alerting.
- Application connection routing to the current primary.
- Network partition and quorum-loss testing.
- Resource sizing and capacity planning.

## 9. Summary

The architecture combines etcd for distributed coordination, Patroni for PostgreSQL high-availability management, and PostgreSQL streaming replication for data synchronization.

Together, these components provide automatic leader election, planned switchover, primary failure handling, and replica recovery.
