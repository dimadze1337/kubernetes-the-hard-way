# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Print the internal IP address and Pod CIDR range for each worker instance:

```bash
{
  NODE_1_IP=$(grep k8s-worker-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep k8s-worker-1 machines.txt | cut -d " " -f 4)
  NODE_2_IP=$(grep k8s-worker-2 machines.txt | cut -d " " -f 1)
  NODE_2_SUBNET=$(grep k8s-worker-2 machines.txt | cut -d " " -f 4)
  NODE_3_IP=$(grep k8s-worker-3 machines.txt | cut -d " " -f 1)
  NODE_3_SUBNET=$(grep k8s-worker-3 machines.txt | cut -d " " -f 4)
}
```

```bash
ssh root@k8s-worker-1 <<EOF
  ip route add ${NODE_2_SUBNET} via ${NODE_2_IP}
  ip route add ${NODE_3_SUBNET} via ${NODE_3_IP}
EOF
```

```bash
ssh root@k8s-worker-2 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
  ip route add ${NODE_3_SUBNET} via ${NODE_3_IP}
EOF
```

```bash
ssh root@k8s-worker-3 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
  ip route add ${NODE_2_SUBNET} via ${NODE_2_IP}
EOF
```

## Verification 

```bash
ssh root@k8s-worker-1 ip route
```

```text
default via 192.168.31.1 dev ens18 proto static 
10.200.1.0/24 via 192.168.31.205 dev ens18 
10.200.2.0/24 via 192.168.31.206 dev ens18 
192.168.31.0/24 dev ens18 proto kernel scope link src 192.168.31.204 
```

```bash
ssh root@k8s-worker-2 ip route
```

```text
default via 192.168.31.1 dev ens18 proto static 
10.200.0.0/24 via 192.168.31.204 dev ens18 
10.200.2.0/24 via 192.168.31.206 dev ens18 
192.168.31.0/24 dev ens18 proto kernel scope link src 192.168.31.205
```

```bash
ssh root@k8s-worker-3 ip route
```

```text
default via 192.168.31.1 dev ens18 proto static 
10.200.0.0/24 via 192.168.31.204 dev ens18 
10.200.1.0/24 via 192.168.31.205 dev ens18 
192.168.31.0/24 dev ens18 proto kernel scope link src 192.168.31.206
```

## We don't want our routes get dissapear after server restart, let's make a systemd service for that

We do this on every machine

```bash
vi /etc/systemd/system/k8s-routes.service
```

```text
[Unit]
Description=Kubernetes pod network routes
Wants=network-online.target
After=network-online.target
After=systemd-networkd.service
Requires=systemd-networkd.service

[Service]
Type=oneshot
ExecStartPre=/usr/bin/bash -c 'until ip addr show ens18 | grep -q "inet "; do sleep 1; done'
ExecStart=/usr/local/bin/k8s-routes.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
vi /usr/local/bin/k8s-routes.sh
```

> Note: Master nodes are tainted with `NoSchedule`, so pods are not scheduled there. Routes are only needed on worker nodes where pods run.

k8s-worker-1
```text
#!/bin/bash

set -e

while ! ip -4 addr show dev ens18 | grep -q 'inet '; do
    sleep 1
done

ip route add 10.200.1.0/24 via 192.168.31.205 || true
ip route add 10.200.2.0/24 via 192.168.31.206 || true
```

k8s-worker-2
```text
#!/bin/bash

set -e

while ! ip -4 addr show dev ens18 | grep -q 'inet '; do
    sleep 1
done

ip route add 10.200.0.0/24 via 192.168.31.204 || true
ip route add 10.200.2.0/24 via 192.168.31.206 || true
```

k8s-worker-3
```text
#!/bin/bash

set -e

while ! ip -4 addr show dev ens18 | grep -q 'inet '; do
    sleep 1
done

ip route add 10.200.0.0/24 via 192.168.31.204 || true
ip route add 10.200.1.0/24 via 192.168.31.205 || true
```

chmod +x /usr/local/bin/k8s-routes.sh

{
  systemctl daemon-reload
  systemctl enable k8s-routes
  systemctl start k8s-routes
}

## Check cluster state

```bash
etcdctl member list

kubectl get lease -n kube-system

kubectl describe lease kube-controller-manager -n kube-system
kubectl describe lease kube-scheduler -n kube-system
```

## Check cluster LeaderElection

Stop api-server on k8s-master-1:

```bash
systemctl stop kube-apiserver
```

From jumpbox check:

```bash
kubectl get pods -A
```

Cluster should keep operating

Next we want to stop kube-controller-manager on current leader:

```bash
systemctl stop kube-controller-manager
```

After few seconds check:

```bash
kubectl get lease -n kube-system
```

```text
root@k8s-master-3:~# kubectl get lease -n kube-system
NAME                                   HOLDER                                                                      AGE
apiserver-c23t2wsviocsvkyhsh7qrdqp7u   apiserver-c23t2wsviocsvkyhsh7qrdqp7u_ca64f874-7130-40b0-a51c-dabfe006d6c3   90m
apiserver-vnwua3ao6dwfps7vqwvywbssce   apiserver-vnwua3ao6dwfps7vqwvywbssce_0de2fe96-aede-41b0-9f34-1d273f895a69   90m
apiserver-zbenbsdlw5akl66fz2mryo2cjm   apiserver-zbenbsdlw5akl66fz2mryo2cjm_441d820a-88d0-4877-a946-1ddf818e1c5a   90m
kube-controller-manager                k8s-master-1_77eef8df-a635-48f7-9fbf-4b8f880c86b9                           90m
kube-scheduler                         k8s-master-1_387b61cd-8cc5-477a-b928-314b77f09d1c                           90m

root@k8s-master-3:~# kubectl get lease -n kube-system
NAME                                   HOLDER                                                                      AGE
apiserver-c23t2wsviocsvkyhsh7qrdqp7u   apiserver-c23t2wsviocsvkyhsh7qrdqp7u_ca64f874-7130-40b0-a51c-dabfe006d6c3   90m
apiserver-vnwua3ao6dwfps7vqwvywbssce   apiserver-vnwua3ao6dwfps7vqwvywbssce_0de2fe96-aede-41b0-9f34-1d273f895a69   90m
apiserver-zbenbsdlw5akl66fz2mryo2cjm   apiserver-zbenbsdlw5akl66fz2mryo2cjm_441d820a-88d0-4877-a946-1ddf818e1c5a   90m
kube-controller-manager                k8s-master-3_338663a4-fc28-46ce-b907-2f2b620f1aeb                           90m
kube-scheduler                         k8s-master-1_387b61cd-8cc5-477a-b928-314b77f09d1c                           90m

```

Leader should change

Next: [Smoke Test](13-smoke-test.md)
