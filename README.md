# PostgreSQL-High-Availability-Cluster-with-Patroni
This document describes the step-by-step setup of a high availability (HA) PostgreSQL cluster using Patroni, HAProxy, and etcd, designed for use in production environments such as monitoring platforms like Zabbix.

Architecture Overview
The cluster setup requires four servers:
1. Zabbix Server: Runs Zabbix and Apache/PHP web services. It does not host a PostgreSQL database.
2. Load Balancer + etcd: Runs HAProxy for traffic routing and also acts as one of the etcd nodes.
3. Database Node 1: Runs PostgreSQL, Patroni, and etcd.
4. Database Node 2: Runs PostgreSQL, Patroni, and etcd.

Traffic Flow
Zabbix communicates with HAProxy on a defined port (e.g., 5000 or 7000). HAProxy checks the health of each PostgreSQL node via HTTP checks on port 8008 to determine which node is the current leader, and forwards traffic accordingly. Only the leader node should handle read/write operations.

1. HAProxy Configuration (Load Balancer Node)
Install HAProxy:
sudo apt install haproxy

Edit /etc/haproxy/haproxy.cfg:
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

 2. etcd Installation (on all 3 etcd nodes)
      Install etcd using binaries:
    wget https://github.com/etcd-io/etcd/releases/download/v3.4.20/etcd-v3.4.20-linux-amd64.tar.gz
    tar -zxvf etcd-v3.4.20-linux-amd64.tar.gz
    mv etcd-v3.4.20-linux-amd64 etcd
    cd etcd
    mv etcd etcdctl /usr/local/bin/



    Create a systemd service:
    
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


Example configuration (/etc/etcd/etcd.conf.yaml):

name: db1
data-dir: /var/lib/etcd
listen-peer-urls: "http://<DB1_IP>:2380"
listen-client-urls: "http://0.0.0.0:2379"
initial-cluster: "db1=http://<DB1_IP>:2380,db2=http://<DB2_IP>:2380,lb=http://<LB_IP>:2380"
initial-cluster-state: "new"
initial-cluster-token: etcd-cluster-token
enable-grpc-gateway: true
enable-v2: true

Start etcd:

systemctl daemon-reexec
systemctl start etcd

3. PostgreSQL & Patroni Setup (DB Nodes)

PostgreSQL Installation
sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
apt-get update
apt-get install postgresql-14

Patroni Installation
apt install python3-venv python3-psycopg2 libpq-dev gcc python3-dev
python3 -m venv myenv
source myenv/bin/activate
pip install setuptools-rust psycopg2-binary
pip install 'patroni[etcd]' 'patroni[raft]'


Patroni Configuration (/etc/patroni/config.yml)

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


Replace <DB1_IP>, <DB2_IP>, and <LB_IP> with your actual server IPs.


4. Initializing the Second Node
On DB Node 2:

pg_basebackup -h <DB1_IP> -D /var/lib/postgresql/14/main/ -U replicator -v -P --wal-method=stream
chown -R postgres:postgres /var/lib/postgresql/14/main/

systemctl start patroni


Troubleshooting
If you encounter missing key errors in etcd, stop the service, delete /var/lib/etcd, and restart.

For permission issues on pg_stat_tmp, run:

chown postgres:postgres /var/run/postgresql/14-main.pg_stat_tmp

If a node shows pending restart, execute:
patronictl -c /etc/patroni/config.yml restart --pending postgres-cluster


Authors
Prepared by: Ali Moradi


