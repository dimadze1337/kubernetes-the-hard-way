# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a single node etcd cluster.

## Prerequisites

Copy `etcd` binaries and systemd unit files to the master machines:

```bash
for host in k8s-master-1 k8s-master-2 k8s-master-3; do
  scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@${host}:~/
done
```

The commands in this lab must be run on the server machine. Login to the server machine using the ssh command. Example:

```bash
ssh root@k8s-master-1
ssh root@k8s-master-2
ssh root@k8s-master-3
```

## Bootstrapping an etcd Cluster

### Install the etcd Binaries

Extract and install the `etcd` server and the `etcdctl` command line utility:

```bash
{
  mv etcd etcdctl /usr/local/bin/
}
```

### Configure the etcd Server

```bash
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-api-server.key kube-api-server.crt \
    /etc/etcd/
}
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:

Create the `etcd.service` systemd unit file:

```bash
mv etcd.service /etc/systemd/system/
```

### Make adjustments for each master server in /etc/systemd/system/etcd.service

```bash
echo > /etc/systemd/system/etcd.service
vi /etc/systemd/system/etcd.service
```
# k8s-master-1
```text
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
--name=k8s-master-1 \
--initial-advertise-peer-urls http://192.168.31.201:2380 \
--listen-peer-urls http://192.168.31.201:2380 \
--listen-client-urls http://192.168.31.201:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://192.168.31.201:2379 \
--initial-cluster-token etcd-cluster-0 \
--initial-cluster=k8s-master-1=http://192.168.31.201:2380,k8s-master-2=http://192.168.31.202:2380,k8s-master-3=http://192.168.31.203:2380 \
--initial-cluster-state new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

# k8s-master-2
```text
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
--name=k8s-master-2 \
--initial-advertise-peer-urls http://192.168.31.202:2380 \
--listen-peer-urls http://192.168.31.202:2380 \
--listen-client-urls http://192.168.31.202:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://192.168.31.202:2379 \
--initial-cluster-token etcd-cluster-0 \
--initial-cluster=k8s-master-1=http://192.168.31.201:2380,k8s-master-2=http://192.168.31.202:2380,k8s-master-3=http://192.168.31.203:2380 \
--initial-cluster-state new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

# k8s-master-3
```text
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
--name=k8s-master-3 \
--initial-advertise-peer-urls http://192.168.31.203:2380 \
--listen-peer-urls http://192.168.31.203:2380 \
--listen-client-urls http://192.168.31.203:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://192.168.31.203:2379 \
--initial-cluster-token etcd-cluster-0 \
--initial-cluster controller=http://127.0.0.1:2380 \
--initial-cluster=k8s-master-1=http://192.168.31.201:2380,k8s-master-2=http://192.168.31.202:2380,k8s-master-3=http://192.168.31.203:2380 \
--initial-cluster-state new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Start the etcd Server

```bash
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```

## Verification

List the etcd cluster members:

```bash
etcdctl member list
```

```text
4015d6d84b047e, started, k8s-master-2, http://192.168.31.202:2380, http://192.168.31.202:2379, false
2210b639171e1176, started, k8s-master-1, http://192.168.31.201:2380, http://192.168.31.201:2379, false
ce204e59ee068817, started, k8s-master-3, http://192.168.31.203:2380, http://192.168.31.203:2379, false
```

```bash
etcdctl \
  --endpoints=http://192.168.31.201:2379,http://192.168.31.202:2379,http://192.168.31.203:2379 \
  member list
  
etcdctl \
  --endpoints=http://192.168.31.201:2379,http://192.168.31.202:2379,http://192.168.31.203:2379 \
  endpoint status --write-out=table
  
etcdctl \
  --endpoints=http://192.168.31.201:2379,http://192.168.31.202:2379,http://192.168.31.203:2379 \
  endpoint health
```

```text
4015d6d84b047e, started, k8s-master-2, http://192.168.31.202:2380, http://192.168.31.202:2379, false
2210b639171e1176, started, k8s-master-1, http://192.168.31.201:2380, http://192.168.31.201:2379, false
ce204e59ee068817, started, k8s-master-3, http://192.168.31.203:2380, http://192.168.31.203:2379, false
+----------------------------+------------------+------------+-----------------+---------+--------+-----------------------+-------+-----------+------------+-----------+------------+--------------------+--------+--------------------------+-------------------+
|          ENDPOINT          |        ID        |  VERSION   | STORAGE VERSION | DB SIZE | IN USE | PERCENTAGE NOT IN USE | QUOTA | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS | DOWNGRADE TARGET VERSION | DOWNGRADE ENABLED |
+----------------------------+------------------+------------+-----------------+---------+--------+-----------------------+-------+-----------+------------+-----------+------------+--------------------+--------+--------------------------+-------------------+
| http://192.168.31.201:2379 | 2210b639171e1176 | 3.6.0-rc.3 |           3.6.0 |   20 kB |  16 kB |                   20% |   0 B |      true |      false |         2 |          9 |                  9 |        |                          |             false |
| http://192.168.31.202:2379 |   4015d6d84b047e | 3.6.0-rc.3 |           3.6.0 |   20 kB |  16 kB |                   20% |   0 B |     false |      false |         2 |          9 |                  9 |        |                          |             false |
| http://192.168.31.203:2379 | ce204e59ee068817 | 3.6.0-rc.3 |           3.6.0 |   20 kB |  16 kB |                   20% |   0 B |     false |      false |         2 |          9 |                  9 |        |                          |             false |
+----------------------------+------------------+------------+-----------------+---------+--------+-----------------------+-------+-----------+------------+-----------+------------+--------------------+--------+--------------------------+-------------------+
http://192.168.31.201:2379 is healthy: successfully committed proposal: took = 4.658953ms
http://192.168.31.202:2379 is healthy: successfully committed proposal: took = 5.454266ms
http://192.168.31.203:2379 is healthy: successfully committed proposal: took = 5.328783ms
```

Next: [Bootstrapping the Kubernetes Control Plane](09-bootstrapping-kubernetes-controllers.md)
