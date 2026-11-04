# PostgreSQL High Availability Lab with Patroni and etcd

A hands-on DevOps lab demonstrating how to build a three-node PostgreSQL high-availability cluster using **Patroni**, **etcd**, and **Ubuntu Server** on VMware.

The project explores PostgreSQL streaming replication, distributed leader election, planned switchover, automatic failover, and replica recovery.

## Project objectives

- Build a three-node virtual machine environment.
- Deploy a three-member etcd cluster for distributed coordination.
- Configure PostgreSQL with Patroni for high availability.
- Establish streaming replication between PostgreSQL nodes.
- Perform a planned primary-node switchover.
- Simulate primary failure and verify automatic failover.
- Restore the failed node and verify that it rejoins as a replica.
- Validate data consistency across all three nodes.

## Architecture

```text
                  PostgreSQL HA Cluster
                     patroni-lab
                          |
                 +------------------+
                 |   etcd Cluster   |
                 |                  |
                 |  etcd1 etcd2 etcd3|
                 +------------------+
                          |
             Patroni leader election
                          |
          +---------------+---------------+
          |               |               |
     +----------+    +----------+    +----------+
     | patroni1 |    | patroni2 |    | patroni3 |
     |          |    |          |    |          |
     | Patroni  |    | Patroni  |    | Patroni  |
     | Postgres |    | Postgres |    | Postgres |
     +----------+    +----------+    +----------+
          |               |               |
          +---------------+---------------+
                  Streaming replication

          One Leader + Two Replicas
```

Each virtual machine runs both etcd and Patroni-managed PostgreSQL.

Patroni uses etcd as its distributed configuration store (DCS) to coordinate cluster membership and leader election.

## Lab environment

| Component | Details |
|---|---|
| Hypervisor | VMware |
| Operating system | Ubuntu Server 26.04.1 LTS |
| Database | PostgreSQL 18.6 |
| HA management | Patroni 4.1.0 |
| Distributed coordination | etcd 3.5.16 |
| Database replication | PostgreSQL streaming replication |
| Network | VMware NAT |
| Number of VMs | 3 |

### Virtual machine topology

| Hostname | Private IP | Services |
|---|---|---|
| patroni1 | 192.168.116.129 | etcd + Patroni + PostgreSQL |
| patroni2 | 192.168.116.132 | etcd + Patroni + PostgreSQL |
| patroni3 | 192.168.116.133 | etcd + Patroni + PostgreSQL |

> These are private lab addresses, not public endpoints. Replace them with your own addresses when adapting the configuration.

## How the cluster works

**etcd**

etcd provides distributed coordination and stores the cluster state used by Patroni. Three members provide quorum-based coordination.

**Patroni**

Patroni manages PostgreSQL instances, monitors the cluster, coordinates leader election, and promotes an eligible replica when the primary becomes unavailable.

**PostgreSQL**

PostgreSQL provides the database service. Replicas continuously receive and replay WAL records from the current primary through streaming replication.

## Implementation and testing

### 1. Cluster bootstrap

- Installed and configured etcd on all three VMs.
- Verified etcd member membership and endpoint health.
- Installed PostgreSQL and Patroni.
- Configured Patroni to use etcd as its DCS.
- Bootstrapped the initial PostgreSQL primary on patroni1.
- Joined patroni2 and patroni3 as replicas.

### 2. Streaming replication

Verified that both replicas were streaming from the primary.

Initial cluster:

| Node | Role | State |
|---|---|---|
| patroni1 | Leader | Running |
| patroni2 | Replica | Streaming |
| patroni3 | Replica | Streaming |

### 3. Planned switchover

Performed a controlled switchover from patroni1 to patroni2 using `patronictl`.

The cluster successfully transferred leadership to patroni2. The other nodes followed the new leader on timeline 2.

### 4. Automatic failover

Simulated primary failure by stopping the Patroni service on patroni2 while leaving etcd running.

Patroni detected the loss of the leader and automatically promoted patroni1.

The cluster advanced to timeline 3.

### 5. Replica recovery

Restarted Patroni on patroni2.

Patroni recognized patroni1 as the current leader and rejoined patroni2 as a replica.

The final cluster state was:

| Node | Role | State | Timeline | Replication lag |
|---|---|---|---|---|
| patroni1 | Leader | Running | 3 | — |
| patroni2 | Replica | Streaming | 3 | 0 |
| patroni3 | Replica | Streaming | 3 | 0 |

## Data consistency verification

Created a test table named `ha_test` on the primary and inserted a record:

```sql
INSERT INTO ha_test (message)
VALUES ('Patroni switchover replication test - 2026-11-04');
```

After the planned switchover, automatic failover, and recovery of patroni2, the same record was successfully queried on all three nodes.

**Verification result:** The test record was present on the leader and both replicas.

## Key skills demonstrated

- Linux server administration
- VMware virtual machine provisioning
- PostgreSQL administration
- Patroni high-availability configuration
- etcd distributed coordination
- PostgreSQL streaming replication
- Leader election and failover
- Database recovery and troubleshooting
- Systemd service management
- SSH-based multi-node administration
- Linux storage and filesystem expansion

## Current limitations and future improvements

This is a learning lab, not a production-ready deployment.

Potential improvements include:

- Configure HAProxy for a stable application connection endpoint.
- Test application reconnection after failover.
- Explore network partition and quorum-loss scenarios.
- Configure automated database backups and recovery testing.
- Add monitoring with Prometheus and Grafana.
- Test controlled failback and additional recovery scenarios.
- Document security hardening, firewall rules, and TLS.

## Learning outcomes

This project provided practical experience building and operating a PostgreSQL high-availability cluster from the ground up.

It demonstrated how distributed coordination, database replication, leader election, automatic failover, and replica recovery work together to improve database availability.

---

**Project type:** Personal DevOps learning lab  
**Status:** Core three-node HA implementation and failover tests completed  
**Environment:** Local VMware virtual machines
