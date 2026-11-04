# Failover Testing – PostgreSQL HA Cluster

## 1. Objective

This document records the high-availability tests performed on the three-node PostgreSQL cluster managed by Patroni and etcd.

The tests demonstrate:

- PostgreSQL streaming replication.
- Planned primary-node switchover.
- Automatic failover following primary failure.
- Recovery of a failed node.
- Data consistency across all three nodes.

## 2. Initial cluster state

Before testing, the cluster was healthy.

| Node | IP address | Role | State |
|---|---|---|---|
| patroni1 | 192.168.116.129 | Leader | Running |
| patroni2 | 192.168.116.132 | Replica | Streaming |
| patroni3 | 192.168.116.133 | Replica | Streaming |

All nodes were on timeline 1, with zero reported replication lag.

Cluster status was checked using:

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml list
```

## 3. Test 1 – Data replication

### Objective

Verify that data written to the primary is replicated to both standby nodes.

### Step 1: Create a test table

Connect to the current primary:

```bash
sudo -u postgres psql -p 5432
```

Create the table:

```sql
CREATE TABLE ha_test (
    id SERIAL PRIMARY KEY,
    message TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Step 2: Insert a test record

```sql
INSERT INTO ha_test (message)
VALUES ('Patroni switchover replication test - 2026-11-04');
```

Verify the record:

```sql
SELECT * FROM ha_test;
```

### Step 3: Check the replicas

Run the same query on patroni2 and patroni3:

```bash
sudo -u postgres psql -p 5432 -c "SELECT * FROM ha_test;"
```

### Result

<badge color="success">Passed</badge>

The test record was visible on both replicas, confirming that streaming replication was working.

## 4. Test 2 – Planned switchover

### Objective

Transfer the primary role from patroni1 to patroni2 without simulating an unexpected failure.

### Step 1: Execute switchover

Run on patroni1:

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml \
  switchover patroni-lab \
  --leader patroni1 \
  --candidate patroni2
```

At the scheduling prompt, press Enter to accept `now`.

At the confirmation prompt, enter `y`.

### Step 2: Verify the new leader

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml list
```

### Observed result

| Node | Role | State | Timeline |
|---|---|---|---|
| patroni1 | Replica | Streaming | 2 |
| patroni2 | Leader | Running | 2 |
| patroni3 | Replica | Streaming | 2 |

### Result

<badge color="success">Passed</badge>

Patroni successfully transferred leadership to patroni2. The other nodes followed the new primary on timeline 2.

## 5. Test 3 – Automatic failover

### Objective

Simulate an unexpected primary failure and verify that Patroni automatically promotes an eligible replica.

### Step 1: Confirm the current primary

Before simulating failure, verify that patroni2 is the leader:

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml list
```

### Step 2: Simulate primary failure

On patroni2, stop the Patroni service:

```bash
sudo systemctl stop patroni
```

This stops Patroni and its managed PostgreSQL instance on patroni2.

The etcd service remains running so that the distributed coordination cluster retains quorum.

### Step 3: Verify the service stopped

```bash
sudo systemctl status patroni --no-pager
```

Expected state:

```text
Active: inactive (dead)
```

### Step 4: Observe automatic promotion

From patroni1, check:

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml list
```

### Observed result

| Node | Role | State | Timeline |
|---|---|---|---|
| patroni1 | Leader | Running | 3 |
| patroni2 | Offline for test | Stopped | — |
| patroni3 | Replica | Streaming | 3 |

### Result

<badge color="success">Passed</badge>

Patroni automatically promoted patroni1 after the previous leader, patroni2, became unavailable.

The cluster advanced from timeline 2 to timeline 3.

## 6. Test 4 – Data integrity after failover

### Objective

Verify that the data written before failover remains available after leadership changes.

On the newly promoted primary, patroni1, run:

```bash
sudo -u postgres psql -p 5432 -c "SELECT * FROM ha_test;"
```

### Observed result

The original test record was present:

```text
id | message
---+--------------------------------------------------
1  | Patroni switchover replication test - 2026-11-04
```

### Result

<badge color="success">Passed</badge>

The existing test record remained accessible on the newly promoted primary.

## 7. Test 5 – Failed node recovery

### Objective

Restore patroni2 and verify that it rejoins the cluster as a replica.

### Step 1: Restart Patroni

On patroni2:

```bash
sudo systemctl start patroni
```

Check the service:

```bash
sudo systemctl status patroni --no-pager
```

### Step 2: Verify its new role

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml list
```

### Observed final state

| Node | Role | State | Timeline | Lag |
|---|---|---|---|---|
| patroni1 | Leader | Running | 3 | — |
| patroni2 | Replica | Streaming | 3 | 0 |
| patroni3 | Replica | Streaming | 3 | 0 |

### Result

<badge color="success">Passed</badge>

Patroni2 successfully recovered and rejoined the cluster as a replica following patroni1.

## 8. Final data verification

The test record was queried independently on all three nodes.

Command:

```bash
sudo -u postgres psql -p 5432 -c "SELECT * FROM ha_test;"
```

Results:

| Node | Test record present |
|---|---|
| patroni1 | Yes |
| patroni2 | Yes |
| patroni3 | Yes |

All three nodes returned the same record.

## 9. Test summary

| Test | Expected outcome | Result |
|---|---|---|
| Streaming replication | Data reaches replicas | Passed |
| Planned switchover | patroni2 becomes leader | Passed |
| Automatic failover | patroni1 becomes leader | Passed |
| Data integrity | Existing test record survives | Passed |
| Replica recovery | patroni2 rejoins as replica | Passed |
| Final cluster health | One leader, two streaming replicas | Passed |

## 10. Important observations

1. Patroni uses etcd to coordinate leader election and maintain cluster state.
2. A planned switchover allows leadership to move to a selected replica.
3. Automatic failover allows an eligible replica to become primary when the existing leader becomes unavailable.
4. PostgreSQL timelines help identify changes in database history after promotion.
5. A failed node can rejoin as a replica after Patroni is restarted, provided it can safely follow the current leader.
6. Replication lag was reported as zero for both replicas in the final cluster status.

## 11. Limitations

These tests were performed in a controlled VMware lab environment.

The results do not establish production-level guarantees for:

- Zero data loss under every failure scenario.
- Application reconnection during failover.
- Network partition handling.
- Recovery after simultaneous node or quorum loss.
- Backup and point-in-time recovery.
- Production security and monitoring.

Further testing is required before using this architecture for production workloads.

## 12. Conclusion

The three-node PostgreSQL cluster successfully demonstrated streaming replication, planned switchover, automatic failover, data integrity verification, and replica recovery.

The tests confirmed that Patroni and etcd can coordinate PostgreSQL high availability across three virtual machines.
