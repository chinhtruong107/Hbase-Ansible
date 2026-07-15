# HBase + HDFS HA Cluster with Ansible

This repository contains Ansible playbooks for deploying a highly available **HDFS** cluster and a distributed **HBase** cluster across multiple Ubuntu nodes. ZooKeeper is used for coordination and automatic failover, JournalNode stores shared edits for NameNode HA, and HBase stores its data on HDFS.

## Cluster Architecture

| Node group | Count | Host examples | Purpose |
|---|---:|---|---|
| ZooKeeper | 3 | `zk1`, `zk2`, `zk3` | Quorum for HDFS ZKFC and HBase |
| NameNode | 2 | `nn1`, `nn2` | HDFS HA Active/Standby pair |
| JournalNode | 3 | `jn1`, `jn2`, `jn3` | Shared edit log storage for NameNode HA |
| DataNode | 3 | `dn1`, `dn2`, `dn3` | HDFS data storage |
| HBase Master | 2 | `hm1`, `hm2` | HBase active/backup masters |
| HBase RegionServer | 3 | `rs1`, `rs2`, `rs3` | HBase region serving |

Total: **16 nodes**.

## Installed Versions

| Component | Version |
|---|---|
| Java | OpenJDK 11 |
| Hadoop | 3.3.6 |
| HBase | 2.5.15, `hadoop3` build |
| ZooKeeper | 3.8.6 |

Software is installed under `/opt`:

- Hadoop: `/opt/hadoop`
- HBase: `/opt/hbase`
- ZooKeeper: `/opt/zookeeper`

Runtime data is stored under `/data`, including:

- ZooKeeper data: `/data/zookeeper`
- NameNode data: `/data/hadoop/namenode`
- DataNode data: `/data/hadoop/datanode`
- JournalNode data: `/data/hadoop/journalnode`

## Repository Layout

```text
Hbase-Ansible/
├── inventory/
│   └── inventory.ini          # Cluster hosts and SSH connection variables
├── group_vars/
│   └── all.yml                # Shared variables, SSH key path, /etc/hosts entries
├── roles/
│   ├── common/                # Java, hadoop user, swap, /etc/hosts
│   ├── zookeeper/             # ZooKeeper installation and startup
│   ├── journalnode/           # Hadoop installation and JournalNode startup
│   ├── namenode/              # Hadoop installation, NameNode/ZKFC format, HA startup
│   ├── datanode/              # Hadoop installation and DataNode startup
│   ├── hbase_master/          # HBase installation and HMaster startup
│   ├── hbase_regionserver/    # HBase installation and RegionServer startup
│   ├── backup/                # HDFS fsimage backup and HBase snapshot creation
│   └── restore/               # HBase table restore from snapshot
├── site.yml                   # Main cluster deployment playbook
├── backup.yml                 # Backup playbook
└── restore.yml                # HBase snapshot restore playbook
```

## Prerequisites

- Ansible installed on the control machine.
- 16 Ubuntu nodes reachable over SSH from the control machine.
- SSH user: `ubuntu`.
- A readable SSH private key, for example `~/.ssh/hbase.pem`.
- Nodes should be in the same VPC/subnet, or at least be able to reach each other by the hostnames defined in `/etc/hosts`.
- Security groups or firewalls must allow internal cluster communication.

Important ports:

| Service | Port |
|---|---:|
| SSH | 22 |
| ZooKeeper client | 2181 |
| NameNode RPC | 8020 |
| NameNode Web UI | 9870 |
| JournalNode | 8485 |
| DataNode Web UI | 9864 |
| HBase Master Web UI | 16010 |
| HBase RegionServer Web UI | 16030 |

## Configure the Inventory

Edit [inventory/inventory.ini](inventory/inventory.ini) with the real node IP addresses:

```ini
[zookeeper]
zk1 ansible_host=172.31.34.199 zk_id=1
zk2 ansible_host=172.31.45.85 zk_id=2
zk3 ansible_host=172.31.41.44 zk_id=3

[namenode]
nn1 ansible_host=172.31.37.50 is_first_namenode=true
nn2 ansible_host=172.31.35.247 is_first_namenode=false

[journalnode]
jn1 ansible_host=172.31.23.79
jn2 ansible_host=172.31.22.210
jn3 ansible_host=172.31.17.216

[datanode]
dn1 ansible_host=172.31.35.51
dn2 ansible_host=172.31.34.67
dn3 ansible_host=172.31.34.197

[hbase_master]
hm1 ansible_host=172.31.37.245
hm2 ansible_host=172.31.39.222

[hbase_regionserver]
rs1 ansible_host=172.31.34.150
rs2 ansible_host=172.31.42.203
rs3 ansible_host=172.31.42.236

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/hbase.pem
```

Then update [group_vars/all.yml](group_vars/all.yml) so that `hosts_entries` matches the inventory. The `common` role writes these entries to `/etc/hosts` on every node.

## Check Connectivity

From the control machine:

```bash
chmod 400 ~/.ssh/hbase.pem
ansible -i inventory/inventory.ini all -m ping
```

If a node is unreachable, check:

- The IP address in the inventory.
- The SSH private key path.
- The SSH user.
- Security group or firewall rules.
- Private key file permissions.

## Deploy the Cluster

Run the main playbook:

```bash
ansible-playbook -i inventory/inventory.ini site.yml
```

Deployment order in `site.yml`:

1. `common`: installs Java, creates the `hadoop` user, disables swap, and updates `/etc/hosts`.
2. `zookeeper`: installs and starts the ZooKeeper ensemble.
3. `journalnode`: installs Hadoop and starts JournalNode.
4. `namenode`: installs Hadoop, formats the first NameNode on `nn1`, formats ZKFC, and starts NameNode/ZKFC.
5. `datanode`: installs Hadoop and starts DataNode.
6. `hbase_master`: installs HBase and starts HMaster.
7. `hbase_regionserver`: installs HBase and starts RegionServer.

## Verify the Deployment

Check HDFS HA state:

```bash
ssh -i ~/.ssh/hbase.pem ubuntu@<nn1-ip> "sudo /opt/hadoop/bin/hdfs haadmin -getAllServiceState"
```

Expected result: one NameNode is `active`, and the other is `standby`.

Check DataNodes:

```bash
ssh -i ~/.ssh/hbase.pem ubuntu@<nn1-ip> "sudo /opt/hadoop/bin/hdfs dfsadmin -report"
```

Check HDFS access:

```bash
ssh -i ~/.ssh/hbase.pem ubuntu@<nn1-ip> "sudo /opt/hadoop/bin/hdfs dfs -ls /"
```

Check HBase:

```bash
ssh -i ~/.ssh/hbase.pem ubuntu@<hm1-ip> "/opt/hbase/bin/hbase shell -n <<< 'status'"
```

Check running Java processes on each node:

```bash
jps
```

Expected processes by node type:

- ZooKeeper: `QuorumPeerMain`
- NameNode: `NameNode`, `DFSZKFailoverController`
- JournalNode: `JournalNode`
- DataNode: `DataNode`
- HBase Master: `HMaster`
- RegionServer: `HRegionServer`

## Backup

[backup.yml](backup.yml) runs on `localhost` and performs the following steps:

- Creates `/backup/<timestamp>` on HDFS.
- Fetches the HDFS fsimage from `nn1`.
- Copies the fsimage to the control machine under `./backups/hdfs_metadata/`.
- Lists HBase tables.
- Creates a snapshot for each HBase table using the name format `<table>_snap_<timestamp>`.

Create the local backup directory before running the playbook:

```bash
mkdir -p backups/hdfs_metadata
```

Run backup:

```bash
ansible-playbook -i inventory/inventory.ini backup.yml
```

List existing HBase snapshots:

```bash
ssh hm1 "/opt/hbase/bin/hbase shell -n <<< 'list_snapshots'"
```

## Restore an HBase Table

[restore.yml](restore.yml) restores one table from an existing HBase snapshot. It requires two variables:

- `table_name`: the table to restore.
- `snapshot_name`: the snapshot to restore from.

Example:

```bash
ansible-playbook -i inventory/inventory.ini restore.yml \
  -e "table_name=test_table snapshot_name=test_table_snap_20260714_060000"
```

Restore flow:

1. Check that the snapshot exists.
2. Disable the table.
3. Restore the snapshot.
4. Enable the table again.
5. Run `count` to verify the table after restore.

## Operational Notes

- Do not reformat NameNode while DataNodes still contain old data. This can cause a `clusterID` mismatch.
- With 3 ZooKeeper nodes or 3 JournalNodes, do not stop more than one node at the same time if you need to preserve quorum.
- Fencing currently uses `shell(true)` in `hdfs-site.xml`, which is suitable for lab/demo environments. Production should use real fencing, such as `sshfence`, to avoid split-brain.
- `downloads.apache.org` may remove older releases. If a download URL returns 404, switch to `https://archive.apache.org/dist/...`.
- If HDFS refuses writes even though disk space appears available, check DataNode reserved/free-space thresholds and actual disk usage.

## Quick Commands

```bash
# Check SSH/Ansible connectivity
ansible -i inventory/inventory.ini all -m ping

# Deploy the full cluster
ansible-playbook -i inventory/inventory.ini site.yml

# Back up HDFS metadata and HBase snapshots
ansible-playbook -i inventory/inventory.ini backup.yml

# Restore one HBase table
ansible-playbook -i inventory/inventory.ini restore.yml \
  -e "table_name=<table> snapshot_name=<snapshot>"
```
