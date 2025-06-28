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

**Traffic Flow**  
Zabbix sends traffic to HAProxy on ports like `5000`, `5001`, or `5002`. HAProxy uses HTTP health checks (`/master`, `/replica`) on port `8008` to determine the **Patroni leader node**, and forwards read/write traffic accordingly.

> ⚠️ Only the **leader** node supports read/write access. Zabbix **must not** connect to replica nodes.

---

## 1️⃣ HAProxy Configuration (Load Balancer Node)

### Install HAProxy

```bash
sudo apt install haproxy


Example /etc/haproxy/haproxy.cfg
haproxy
Copy
Edit

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
