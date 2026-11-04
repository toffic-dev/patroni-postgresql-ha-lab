# Installation Guide – PostgreSQL HA Cluster with Patroni and etcd

## 1. Introduction

This guide documents the installation and initial configuration of a three-node PostgreSQL high-availability cluster using Patroni and etcd.

The lab was implemented using three Ubuntu Server virtual machines running on VMware.

## 2. Prerequisites

### Hardware

- A computer capable of running three virtual machines.
- VMware installed.
- Sufficient memory and storage for the VMs.

### Software

- Ubuntu Server 26.04.1 LTS on all three VMs.
- etcd 3.5.16.
- PostgreSQL 18.6.
- Patroni 4.1.0.
- SSH access between all nodes.

## 3. Virtual machine preparation

Three virtual machines were created.

| Hostname | IP address | Operating system |
|---|---|---|
| patroni1 | 192.168.116.129 | Ubuntu Server 26.04.1 |
| patroni2 | 192.168.116.132 | Ubuntu Server 26.04.1 |
| patroni3 | 192.168.116.133 | Ubuntu Server 26.04.1 |

The machines were connected through a VMware NAT network.

### Configure host resolution

On all three nodes, edit:

```bash
sudo nano /etc/hosts
```

Add:

```text
192.168.116.129 patroni1
192.168.116.132 patroni2
192.168.116.133 patroni3
```

Verify connectivity:

```bash
ping -c 3 patroni1
ping -c 3 patroni2
ping -c 3 patroni3
```

## 4. Install etcd

Install etcd on all three nodes:

```bash
sudo apt update
sudo apt install etcd-server etcd-client -y
```

Verify the installation:

```bash
etcd --version
etcdctl version
```

### Configure etcd

Each node requires a unique name and IP address.

The configuration is stored in:

```text
/etc/default/etcd
```

Important settings include:

- `ETCD_NAME`
- `ETCD_DATA_DIR`
- `ETCD_INITIAL_CLUSTER`
- `ETCD_INITIAL_CLUSTER_STATE`
- `ETCD_INITIAL_CLUSTER_TOKEN`
- `ETCD_LISTEN_CLIENT_URLS`
- `ETCD_ADVERTISE_CLIENT_URLS`
- `ETCD_LISTEN_PEER_URLS`
- `ETCD_INITIAL_ADVERTISE_PEER_URLS`

The initial cluster contains all three members:

```text
patroni1=http://192.168.116.129:2380
patroni2=http://192.168.116.132:2380
patroni3=http://192.168.116.133:2380
```

Use the appropriate node name and client/peer addresses for each machine.

Start and enable etcd:

```bash
sudo systemctl enable etcd
sudo systemctl restart etcd
sudo systemctl status etcd --no-pager
```

### Verify etcd health

Run from any node:

```bash
export ETCDCTL_API=3

etcdctl \
  --endpoints=http://192.168.116.129:2379,http://192.168.116.132:2379,http://192.168.116.133:2379 \
  endpoint health
```

Verify membership:

```bash
etcdctl \
  --endpoints=http://192.168.116.129:2379,http://192.168.116.132:2379,http://192.168.116.133:2379 \
  member list
```

Expected result: three started etcd members and healthy endpoints.

## 5. Install PostgreSQL and Patroni

Perform these steps on all three nodes.

Update package information:

```bash
sudo apt update
```

Install PostgreSQL, Patroni, and the required dependencies:

```bash
sudo apt install postgresql postgresql-client patroni python3-etcd python3-psycopg2 -y
```

Verify the versions:

```bash
psql --version
patroni --version
```

### Install the Python etcd dependency

Verify that Patroni can load its etcd integrations:

```bash
sudo -u postgres python3 -c "import etcd; print('python-etcd: OK')"
sudo -u postgres python3 -c "import patroni.dcs.etcd3; print('Patroni etcd3: OK')"
```

## 6. Prepare PostgreSQL for Patroni management

Ubuntu creates a default PostgreSQL cluster during installation.

The default cluster was preserved, while Patroni was configured to use a separate data directory.

Mask the standalone PostgreSQL service on all nodes:

```bash
sudo systemctl stop postgresql
sudo systemctl mask postgresql
```

Create the Patroni data directory:

```bash
sudo install -d -o postgres -g postgres -m 700 \
  /var/lib/postgresql/18/patroni
```

Create the configuration directory:

```bash
sudo install -d -o postgres -g postgres -m 750 /etc/patroni
```

## 7. Configure Patroni

Create the configuration file on each node:

```bash
sudo -u postgres nano /etc/patroni/config.yml
```

Each configuration must contain:

- A unique node name.
- Its own REST API address.
- Its own PostgreSQL connection address.
- The common etcd member addresses.
- The same cluster scope and namespace.
- PostgreSQL authentication settings.
- Bootstrap settings and replication parameters.

The configuration should use:

```yaml
scope: patroni-lab
namespace: /service/
```

The PostgreSQL data directory:

```yaml
data_dir: /var/lib/postgresql/18/patroni
```

The PostgreSQL binary directory:

```yaml
bin_dir: /usr/lib/postgresql/18/bin
```

**Important:** Do not copy one node's complete configuration unchanged to the other nodes. Update the node-specific name and IP addresses.

Set secure permissions:

```bash
sudo chown postgres:postgres /etc/patroni/config.yml
sudo chmod 600 /etc/patroni/config.yml
```

Validate the configuration:

```bash
sudo -u postgres patroni --validate-config /etc/patroni/config.yml
```

## 8. Start Patroni

Enable Patroni on all three nodes:

```bash
sudo systemctl enable patroni
```

Start Patroni on the initial bootstrap node first:

```bash
sudo systemctl start patroni
```

Check the service:

```bash
sudo systemctl status patroni --no-pager
```

Once patroni1 has initialized the cluster, start Patroni on patroni2 and patroni3:

```bash
sudo systemctl start patroni
```

Check the cluster:

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml list
```

Expected initial topology:

| Node | Role | State |
|---|---|---|
| patroni1 | Leader | Running |
| patroni2 | Replica | Streaming |
| patroni3 | Replica | Streaming |

## 9. Verify streaming replication

On the primary, connect to PostgreSQL:

```bash
sudo -u postgres psql -p 5432
```

Check connected replicas:

```sql
SELECT application_name, client_addr, state, sync_state
FROM pg_stat_replication;
```

On a replica, verify recovery mode:

```sql
SELECT pg_is_in_recovery();
```

Expected result:

```text
t
```

This indicates that the node is operating as a standby.

## 10. Troubleshooting

### Patroni cannot load etcd integration

Check the Python dependency:

```bash
sudo -u postgres python3 -c "import etcd"
```

### Check Patroni service logs

```bash
sudo journalctl -u patroni -n 100 --no-pager
```

### Check etcd service logs

```bash
sudo journalctl -u etcd -n 100 --no-pager
```

### Check etcd endpoint health

```bash
export ETCDCTL_API=3

etcdctl \
  --endpoints=http://192.168.116.129:2379,http://192.168.116.132:2379,http://192.168.116.133:2379 \
  endpoint health
```

### Check cluster status

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml list
```

## 11. Important notes

- Keep etcd running while testing PostgreSQL failover.
- Do not start PostgreSQL independently while Patroni manages the instance.
- Never delete the Patroni data directory as a troubleshooting shortcut.
- Keep credentials out of Git.
- Back up configuration files before changing them.
- Verify cluster health before performing a switchover or failover test.

## 12. Completion

At the end of this installation process, the cluster should contain one PostgreSQL leader and two streaming replicas, coordinated by a three-member etcd cluster.
