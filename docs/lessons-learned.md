# Lessons Learned and Troubleshooting

## 1. Introduction

Building the three-node PostgreSQL high-availability cluster involved more than installing software. Several infrastructure and configuration challenges had to be identified and resolved.

This document records the main issues encountered, their causes, the solutions applied, and the technical lessons learned.

## 2. VMware disk capacity limitations

### Problem

The initial virtual machine disks were approximately 12 GB. As PostgreSQL, etcd, and supporting packages were installed, available disk space became insufficient.

### Investigation

Checked filesystem usage:

```bash
df -h
```

Inspected block devices and partitions:

```bash
lsblk
```

### Solution

Increased the VMware virtual disk size to 25 GB on each VM.

Then expanded the Linux partition:

```bash
sudo growpart /dev/sda 3
```

Resized the LVM physical volume:

```bash
sudo pvresize /dev/sda3
```

Extended the root logical volume and filesystem:

```bash
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

Verified the result:

```bash
df -h /
```

### Lesson learned

Increasing a virtual disk's size in VMware does not automatically expand the guest operating system's filesystem. Partition and LVM resizing may also be required.

## 3. Patroni could not load the etcd integration

### Problem

During the initial Patroni startup, the service failed to identify a suitable distributed configuration store.

The logs reported errors related to importing Patroni's etcd integrations.

### Investigation

Checked Patroni logs:

```bash
sudo journalctl -u patroni -n 100 --no-pager
```

Checked Python module availability:

```bash
sudo -u postgres python3 -c "import etcd"
```

### Root cause

The required Python etcd integration dependency was missing.

### Solution

Installed the required packages:

```bash
sudo apt install python3-etcd python3-dnspython -y
```

Verified the integration:

```bash
sudo -u postgres python3 -c "import etcd; print('python-etcd: OK')"

sudo -u postgres python3 -c \
  "import patroni.dcs.etcd3; print('Patroni etcd3: OK')"
```

Validated Patroni configuration:

```bash
sudo -u postgres patroni --validate-config /etc/patroni/config.yml
```

### Lesson learned

Installing the main application package does not always install every optional integration dependency. Dependency validation should be performed before starting a distributed service.

## 4. Separating the default PostgreSQL cluster from Patroni

### Problem

Ubuntu automatically created a PostgreSQL cluster named `18/main` during package installation.

Starting PostgreSQL independently could conflict with Patroni's management of PostgreSQL.

### Solution

Preserved the default PostgreSQL data directory and configured Patroni to use a separate data directory:

```text
/var/lib/postgresql/18/patroni
```

Masked the standalone PostgreSQL service:

```bash
sudo systemctl stop postgresql
sudo systemctl mask postgresql
```

### Lesson learned

When using Patroni, PostgreSQL lifecycle management should be delegated to Patroni. Existing database data should be preserved rather than removed unnecessarily.

## 5. Understanding etcd quorum

### Challenge

A three-member etcd cluster requires a majority of members to remain available for quorum.

### Investigation

Verified endpoint health:

```bash
export ETCDCTL_API=3

etcdctl \
  --endpoints=http://192.168.116.129:2379,http://192.168.116.132:2379,http://192.168.116.133:2379 \
  endpoint health
```

### Lesson learned

Three etcd members can tolerate one member failure while maintaining quorum. Losing two members prevents the remaining member from maintaining quorum.

The automatic failover exercise deliberately stopped Patroni on one node while leaving etcd running.

## 6. Understanding planned switchover versus automatic failover

### Challenge

A planned switchover and an automatic failover both change the PostgreSQL primary, but they are different operations.

### Solution and observations

**Planned switchover**

Used `patronictl switchover` to transfer leadership from patroni1 to patroni2.

**Automatic failover**

Stopped the Patroni service on patroni2 while it was primary.

Patroni promoted patroni1 automatically.

### Lesson learned

A planned switchover is an operator-initiated operation performed while coordinating a controlled transition. Automatic failover is triggered when the current leader becomes unavailable and an eligible replica can be promoted.

## 7. Recovering a failed node

### Challenge

After automatic failover, patroni2 was no longer the primary.

It needed to return to the cluster without creating a second primary.

### Solution

Restarted Patroni:

```bash
sudo systemctl start patroni
```

Checked the logs:

```bash
sudo journalctl -u patroni -n 100 --no-pager
```

The logs confirmed:

- patroni1 remained the lock owner.
- patroni2 identified itself as a secondary.
- WAL streaming resumed on timeline 3.

### Lesson learned

Patroni can manage a returning former primary and reconfigure it to follow the current leader when recovery conditions permit.

## 8. Resource constraints in a small VM lab

### Challenge

Each VM had approximately 1.6 GiB of RAM. Memory pressure and swap usage were significant during the exercises.

### Solution

Used conservative PostgreSQL memory settings, avoided unnecessary workloads, and checked available memory during testing.

Commands used:

```bash
free -h
```

```bash
ps aux --sort=-%mem | head
```

### Lesson learned

A successful cluster configuration does not automatically mean the infrastructure is adequately sized. Resource monitoring is important even in a small learning environment.

## 9. Key technical lessons

The project reinforced several important concepts:

1. **Infrastructure provisioning:** Virtual disk allocation and guest filesystem resizing are separate operations.
2. **Dependency management:** Optional integrations must be checked explicitly.
3. **Service ownership:** Patroni should control PostgreSQL lifecycle operations.
4. **Distributed coordination:** etcd quorum is essential for reliable leader election.
5. **Replication:** PostgreSQL streaming replication keeps standby databases updated.
6. **Failover:** A healthy replica can be promoted when the primary becomes unavailable.
7. **Recovery:** A former primary must rejoin the current cluster safely.
8. **Observability:** Service logs and cluster status are essential troubleshooting tools.
9. **Resource planning:** Memory, swap, CPU, and storage affect database stability.

## 10. Future improvements

Potential next steps include:

- Deploy HAProxy for primary connection routing.
- Test application reconnection after failover.
- Test network partitions and quorum loss safely.
- Configure database backups and point-in-time recovery.
- Add Prometheus and Grafana monitoring.
- Explore TLS and authentication for etcd and PostgreSQL.
- Evaluate resource requirements for production deployment.

## 11. Conclusion

The troubleshooting process was an important part of this project.

Resolving storage, dependency, service-management, and recovery issues provided practical experience operating a distributed PostgreSQL high-availability environment.
