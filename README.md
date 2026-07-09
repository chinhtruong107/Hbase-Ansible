# HBase + HDFS HA Cluster — Ansible Automation

Ansible playbooks to deploy a fully highly-available **HDFS** and **HBase** cluster on AWS EC2, backed by an external **ZooKeeper** ensemble. Includes an automated **HA test suite** to validate failover behavior after deployment.

## Architecture

| Component | Nodes | Role |
|---|---|---|
| ZooKeeper | 3 nodes (`zk1-zk3`) | Coordination service, used by HDFS ZKFC and HBase |
| NameNode | 2 nodes (`nn1`, `nn2`) | HDFS HA (Active/Standby) via QJM + automatic failover (ZKFC) |
| JournalNode | 3 nodes (`jn1-jn3`) | Shared edits storage for NameNode HA |
| DataNode | 3 nodes (`dn1-dn3`) | HDFS data storage |
| HMaster | 2 nodes (`hm1`, `hm2`) | HBase Master (Active/Backup) |
| RegionServer | 3 nodes (`rs1-rs3`) | HBase RegionServer |

**Total: 16 EC2 instances.**

Software versions:
- Ubuntu 22.04/24.04
- Hadoop 3.3.6
- HBase 2.5.15 (hadoop3 build)
- ZooKeeper 3.8.6

## Repository Structure

```
Hbase-Ansible/
├── inventory/
│   └── inventory.ini          # Hosts, groups, and connection vars
├── group_vars/
│   └── all.yml                # Shared vars (e.g. /etc/hosts entries)
├── roles/
│   ├── common/                # Java, hadoop user, /etc/hosts, swap
│   ├── zookeeper/              # ZooKeeper ensemble
│   ├── journalnode/             # HDFS JournalNode
│   ├── namenode/               # HDFS NameNode HA + ZKFC
│   ├── datanode/               # HDFS DataNode
│   ├── hbase_master/            # HBase HMaster (active/backup)
│   ├── hbase_regionserver/      # HBase RegionServer
│   └── ha_test/                # Automated HA failure/failover test suite
├── site.yml                    # Main deployment playbook
└── ha_test.yml                 # HA test suite playbook
```

## Prerequisites

- 16 EC2 instances (see sizing recommendation below), all in the same VPC/subnet, reachable from an Ansible control node.
- A shared Security Group with a self-referencing "allow all" rule (simplest option for a lab/test cluster) plus SSH (22) open from the control node.
- One SSH key pair (`.pem`) present on the control node, used to reach every managed node.
- Ansible installed on the control node.

### Recommended instance sizing

| Node type | Instance type | Reason |
|---|---|---|
| ZooKeeper | t3.small | Needs to stay stable — quorum leader election |
| NameNode | t3.medium | Manages cluster metadata, higher RAM |
| HMaster | t3.medium | Manages cluster metadata, higher RAM |
| JournalNode | t2.micro | Lightweight, only writes shared edit logs |
| DataNode | t3.small | Handles real read/write I/O |
| RegionServer | t3.small | Handles real read/write I/O |

> Disk: at least 20 GB per node is recommended. Hadoop refuses to write once free space drops below its safety margin, even if the disk isn't technically full.

## Setup

### 1. Configure the inventory

Edit `inventory/inventory.ini` with your real instance IPs:

```ini
[zookeeper]
zk1 ansible_host=<ip>
zk2 ansible_host=<ip>
zk3 ansible_host=<ip>

[namenode]
nn1 ansible_host=<ip> is_first_namenode=true
nn2 ansible_host=<ip> is_first_namenode=false

[journalnode]
jn1 ansible_host=<ip>
jn2 ansible_host=<ip>
jn3 ansible_host=<ip>

[datanode]
dn1 ansible_host=<ip>
dn2 ansible_host=<ip>
dn3 ansible_host=<ip>

[hbase_master]
hm1 ansible_host=<ip>
hm2 ansible_host=<ip>

[hbase_regionserver]
rs1 ansible_host=<ip>
rs2 ansible_host=<ip>
rs3 ansible_host=<ip>

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/your-key.pem
```

### 2. Copy the SSH key to every node (if not baked into the AMI)

```bash
chmod 400 ~/.ssh/your-key.pem
```

### 3. Deploy the cluster

```bash
ansible-playbook -i inventory/inventory.ini site.yml
```

Deployment order (handled automatically by `site.yml`):
1. `common` — Java, hadoop user, `/etc/hosts`, swap
2. `zookeeper` — ensemble bootstrap
3. `journalnode` — Hadoop install + JournalNode start
4. `namenode` — Hadoop install, format (first node only), NameNode + ZKFC start
5. `datanode` — Hadoop install + DataNode start
6. `hbase_master` — HBase install + HMaster start
7. `hbase_regionserver` — HBase install + RegionServer start

### 4. Verify the cluster

```bash
# HDFS HA state (expect 1 active, 1 standby)
ssh -i ~/.ssh/your-key.pem ubuntu@<nn1-ip> "sudo /opt/hadoop/bin/hdfs haadmin -getAllServiceState"

# DataNodes registered (expect 3)
ssh -i ~/.ssh/your-key.pem ubuntu@<nn1-ip> "sudo /opt/hadoop/bin/hdfs dfsadmin -report"

# HBase cluster status (expect: 1 active master, 1 backup masters, 3 servers, 0 dead)
ssh -i ~/.ssh/your-key.pem ubuntu@<hm1-ip> "/opt/hbase/bin/hbase shell <<< 'status'"
```

## HA Test Suite

The `ha_test` role automates the manual HA validation test cases (based on the internal HA test document), covering:

| Tag | Test case | What it validates |
|---|---|---|
| `test1` | NameNode failover | Standby NameNode takes over when Active is stopped |
| `test2` | ZKFC failover | ZooKeeper Failover Controller triggers automatic failover |
| `test3` | JournalNode quorum | HDFS keeps working after losing 1 of 3 JournalNodes |
| `test4` | ZooKeeper quorum | HDFS/HBase keep working after losing 1 of 3 ZooKeeper nodes |
| `test5` | HMaster failover | Backup HMaster takes over when Active HMaster is stopped |
| `test6` | RegionServer/DataNode failure | Cluster tolerates the loss of a single RegionServer/DataNode |

Each test captures a baseline, injects the failure, checks the expected outcome, and restores the cluster to a healthy state before finishing.

Run the full suite:

```bash
ansible-playbook -i inventory/inventory.ini ha_test.yml
```

Run a single test case:

```bash
ansible-playbook -i inventory/inventory.ini ha_test.yml --tags test1
```

> **Rules:** only fail one component at a time. Never stop both NameNodes, or 2+ JournalNodes, or 2+ ZooKeeper nodes simultaneously — this breaks quorum and the cluster will not recover automatically. Always restore the cluster to a healthy baseline before moving to the next test case.

## Fencing Configuration Note

HDFS HA fencing (`dfs.ha.fencing.methods = sshfence`) requires a working passwordless SSH connection **between the NameNode hosts, as the same OS user that runs the NameNode process** (here: `root`, since Ansible tasks run with `become: yes`). Make sure `/root/.ssh/id_rsa` exists and is exchanged between `nn1` and `nn2`, or fencing will silently fail and the cluster will rely on Hadoop's fallback behavior instead of a real fence — which is not safe against split-brain in a real failure scenario (as opposed to a clean stop).

## Known Gotchas

- Apache mirrors (`downloads.apache.org`) only keep the last 1–2 patch releases of each project; pin download URLs to `archive.apache.org/dist/...` for a version that won't 404 on you later.
- `hdfs namenode -format` and `hdfs zkfc -formatZK` prompt for confirmation if already formatted — use `-force` / `-nonInteractive` and treat exit code 2 (zkfc) as "already formatted, not an error" in automation.
- Disk usage above ~90% will make DataNodes refuse writes even though `df` shows free space — this manifests as `could only be written to 0 of the 1 minReplication nodes`.
- A NameNode/DataNode `clusterID` mismatch (usually from re-formatting the NameNode without wiping DataNode storage) will keep DataNodes from registering; wipe `/data/hadoop/datanode` on all DataNodes and restart them to fix it.
