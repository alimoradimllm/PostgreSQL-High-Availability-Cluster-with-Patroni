![fee5a069-453b-4e67-bd83-70710104a4d3](https://github.com/user-attachments/assets/08d6ff64-cbc1-4101-8746-a4813210cc36)
# PostgreSQL High Availability Cluster with Patroni

This guide provides step-by-step instructions for deploying a **high availability PostgreSQL cluster** using **Patroni**, **etcd**, and **HAProxy**, designed for production environments such as monitoring platforms like **Zabbix**.

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [HAProxy Setup](#1-haproxy-configuration-load-balancer-node)
3. [etcd Installation](#2-etcd-installation-on-all-3-etcd-nodes)
4. [PostgreSQL & Patroni Setup](#3-postgresql--patroni-setup-db-nodes)
5. [Initializing the Second Node](#4-initializing-the-second-node)
6. [Troubleshooting](#troubleshooting)
7. [Authors](#authors)

---

## 🏗 Architecture Overview

The cluster consists of **four servers**:

| Role              | Components                            |
|-------------------|----------------------------------------|
| Zabbix Server     | Zabbix + Apache/PHP                    |
| Load Balancer     | HAProxy + etcd                         |
| Database Node 1   | PostgreSQL + Patroni + etcd            |
| Database Node 2   | PostgreSQL + Patroni + etcd            |

![ChatGPT Image Jun 28, 2025, 03_16_05 PM](https://github.com/user-attachments/assets/796db09b-136b-4661-a8aa-67ca3a5a496a)


**Traffic Flow**  
Zabbix sends traffic to HAProxy on ports like `5000`, `5001`, or `5002`. HAProxy uses HTTP health checks (`/master`, `/replica`) on port `8008` to determine the **Patroni leader node**, and forwards read/write traffic accordingly.

> ⚠️ Only the **leader** node supports read/write access. Zabbix **must not** connect to replica nodes.

---

## 1️⃣ HAProxy Configuration (Load Balancer Node)

### Install HAProxy

```bash
sudo apt install haproxy
```

### Example `/etc/haproxy/haproxy.cfg`

```haproxy
global
    maxconn 1000

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind 0.0.0.0:5000
    stats enable
    stats uri /

listen production
    bind 0.0.0.0:5002
    option httpchk OPTIONS /master
    http-check expect status 200
    server pg_node1 <DB1_IP>:5432 check port 8008
    server pg_node2 <DB2_IP>:5432 check port 8008

listen standby
    bind 0.0.0.0:5001
    option httpchk OPTIONS /replica
    http-check expect status 200
    server pg_node1 <DB1_IP>:5432 check port 8008
    server pg_node2 <DB2_IP>:5432 check port 8008
```

---

## 2️⃣ etcd Installation (on all 3 etcd nodes)

### Install from Binary

```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.4.20/etcd-v3.4.20-linux-amd64.tar.gz
tar -zxvf etcd-v3.4.20-linux-amd64.tar.gz
mv etcd-v3.4.20-linux-amd64 etcd
cd etcd
sudo mv etcd etcdctl /usr/local/bin/
```

### Create systemd Service

```ini
# /etc/systemd/system/etcd.service
[Unit]
Description=ETCD
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.conf.yaml
Restart=always
Type=simple

[Install]
WantedBy=multi-user.target
```

### Sample Configuration `/etc/etcd/etcd.conf.yaml`

```yaml
name: db1
data-dir: /var/lib/etcd
listen-peer-urls: "http://<DB1_IP>:2380"
listen-client-urls: "http://0.0.0.0:2379"
initial-cluster: "db1=http://<DB1_IP>:2380,db2=http://<DB2_IP>:2380,lb=http://<LB_IP>:2380"
initial-cluster-state: "new"
initial-cluster-token: etcd-cluster-token
enable-grpc-gateway: true
enable-v2: true
```

### Start Service

```bash
systemctl daemon-reexec
systemctl start etcd
```

---

## 3️⃣ PostgreSQL & Patroni Setup (DB Nodes)

### Install PostgreSQL 14

```bash
sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
apt-get update
apt-get install postgresql-14
```

### Install Patroni

```bash
apt install python3-venv python3-psycopg2 libpq-dev gcc python3-dev
python3 -m venv myenv
source myenv/bin/activate
pip install setuptools-rust psycopg2-binary
pip install 'patroni[etcd]' 'patroni[raft]'
```

### Sample Patroni Config `/etc/patroni/config.yml`

```yaml
scope: postgres-cluster
name: db1
namespace: /db/

restapi:
  listen: <DB1_IP>:8008
  connect_address: <DB1_IP>:8008

etcd:
  host: <DB1_IP>:2379

bootstrap:
  dcs:
    ttl: 60
    loop_wait: 20
    retry_timeout: 10
    maximum_lag_on_failover: 2097152
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 256
        max_standby_streaming_delay: '30s'
        checkpoint_timeout: '60min'
        max_wal_size: '8GB'
        synchronous_commit: 'remote_apply'
        wal_compression: on
  initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8

postgresql:
  listen: <DB1_IP>:5432
  connect_address: <DB1_IP>:5432
  data_dir: /var/lib/postgresql/14/main
  config_dir: /etc/postgresql/14/main
  bin_dir: /usr/lib/postgresql/14/bin
  authentication:
    superuser:
      username: postgres
      password: z145
    replication:
      username: replicator
      password: replicator_z145

watchdog:
  mode: automatic
```

> ✅ Replace `<DB1_IP>`, `<DB2_IP>`, and `<LB_IP>` with your actual server IPs.

---

## 4️⃣ Initializing the Second Node

On **DB Node 2**:

```bash
pg_basebackup -h <DB1_IP> -D /var/lib/postgresql/14/main/ -U replicator -v -P --wal-method=stream
chown -R postgres:postgres /var/lib/postgresql/14/main/
systemctl start patroni
```

---

## 🛠 Troubleshooting

| Issue                               | Solution                                                                   |
|------------------------------------|----------------------------------------------------------------------------|
| **Missing etcd key error**         | Stop etcd, delete `/var/lib/etcd`, and restart the service                 |
| **pg_stat_tmp permission denied**  | `chown postgres:postgres /var/run/postgresql/14-main.pg_stat_tmp`          |
| **Pending restart in Patroni**     | `patronictl -c /etc/patroni/config.yml restart --pending postgres-cluster` |

---

## 👨‍💻 Authors

Prepared by **Ali Moradi**  

