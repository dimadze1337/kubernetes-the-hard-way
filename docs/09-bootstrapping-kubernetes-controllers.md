# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane. The following components will be installed on the master machines: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

Connect to the `jumpbox` and copy Kubernetes binaries and systemd unit files to the master machines:

```bash
for host in k8s-master-1 k8s-master-2 k8s-master-3; do
  scp \
  downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  units/kube-apiserver.service \
  units/kube-controller-manager.service \
  units/kube-scheduler.service \
  configs/kube-scheduler.yaml \
  configs/kube-apiserver-to-kubelet.yaml \
  root@${host}:~/
done
```

The commands in this lab must be run on the master machines. Login to the master machines using the `ssh` command. Example:

```bash
ssh root@k8s-master-1
ssh root@k8s-master-2
ssh root@k8s-master-3
```

## Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory:

```bash
mkdir -p /etc/kubernetes/config
```

### Install the Kubernetes Controller Binaries

Install the Kubernetes binaries:

```bash
{
  mv kube-apiserver \
    kube-controller-manager \
    kube-scheduler kubectl \
    /usr/local/bin/
}
```

### Configure the Kubernetes API Server

```bash
{
  mkdir -p /var/lib/kubernetes/

  mv ca.crt ca.key \
    kube-api-server.key kube-api-server.crt \
    service-accounts.key service-accounts.crt \
    encryption-config.yaml \
    /var/lib/kubernetes/
}
```

Modify a bit `kube-apiserver.service`, we should change `--advertise-address` and `--etcd-servers`

For every master take his own ip
```text
--advertise-address=192.168.31.201
--advertise-address=192.168.31.202
--advertise-address=192.168.31.203
```

for k8s-master-1
```text
--advertise-address=192.168.31.201 \
--etcd-servers=http://192.168.31.201:2379,http://192.168.31.202:2379,http://192.168.31.203:2379 \
```

Modify a bit `kube-controller-manager.service`

```text
  --leader-elect=true \
  --service-cluster-ip-range=10.0.0.0/24 \

```


Create the `kube-apiserver.service` systemd unit file:

```bash
mv kube-apiserver.service \
  /etc/systemd/system/kube-apiserver.service
```

### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into place:

```bash
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create the `kube-controller-manager.service` systemd unit file:

```bash
mv kube-controller-manager.service /etc/systemd/system/
```

### Configure the Kubernetes Scheduler

Move the `kube-scheduler` kubeconfig into place:

```bash
mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create the `kube-scheduler.yaml` configuration file:

```bash
mv kube-scheduler.yaml /etc/kubernetes/config/
```

Create the `kube-scheduler.service` systemd unit file:

```bash
mv kube-scheduler.service /etc/systemd/system/
```

Or just all in one
```bash
mv kube-apiserver.service \
  /etc/systemd/system/kube-apiserver.service

mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

mv kube-controller-manager.service /etc/systemd/system/

mv kube-scheduler.kubeconfig /var/lib/kubernetes/

mv kube-scheduler.yaml /etc/kubernetes/config/

mv kube-scheduler.service /etc/systemd/system/
```

### Start the Controller Services

```bash
{
  systemctl daemon-reload

  systemctl enable kube-apiserver \
    kube-controller-manager kube-scheduler

  systemctl start kube-apiserver \
    kube-controller-manager kube-scheduler
}
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

You can check if any of the control plane components are active using the `systemctl` command. For example, to check if the `kube-apiserver` fully initialized, and active, run the following command:

```bash
systemctl is-active kube-apiserver
```

For a more detailed status check, which includes additional process information and log messages, use the `systemctl status` command:

```bash
systemctl status kube-apiserver
```

If you run into any errors, or want to view the logs for any of the control plane components, use the `journalctl` command. For example, to view the logs for the `kube-apiserver` run the following command:

```bash
journalctl -u kube-apiserver
```

### Verification

At this point the Kubernetes control plane components should be up and running. Verify this using the `kubectl` command line tool:

```bash
kubectl cluster-info \
  --kubeconfig admin.kubeconfig
```

```text
Kubernetes control plane is running at https://127.0.0.1:6443
```

```bash
curl -k https://192.168.31.201:6443/version
curl -k https://192.168.31.202:6443/version
curl -k https://192.168.31.203:6443/version

curl -k https://127.0.0.1:6443/readyz

etcdctl \
  --endpoints=http://192.168.31.201:2379,http://192.168.31.202:2379,http://192.168.31.203:2379 \
  endpoint health


kubectl -n kube-system get lease \
  --kubeconfig admin.kubeconfig
  
kubectl get --raw=/readyz \
  --kubeconfig admin.kubeconfig
```

Output
```text

{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  "gitCommit": "32cc146f75aad04beaaa245a7157eb35063a9f99",
  "gitTreeState": "clean",
  "buildDate": "2025-03-11T19:52:21Z",
  "goVersion": "go1.23.6",
  "compiler": "gc",
  "platform": "linux/amd64"
}

ok

http://192.168.31.201:2379 is healthy: successfully committed proposal: took = 9.58698ms
http://192.168.31.202:2379 is healthy: successfully committed proposal: took = 9.37791ms
http://192.168.31.203:2379 is healthy: successfully committed proposal: took = 9.545008ms
NAME                                   HOLDER                                                                      AGE
apiserver-c23t2wsviocsvkyhsh7qrdqp7u   apiserver-c23t2wsviocsvkyhsh7qrdqp7u_ca64f874-7130-40b0-a51c-dabfe006d6c3   4m4s
apiserver-vnwua3ao6dwfps7vqwvywbssce   apiserver-vnwua3ao6dwfps7vqwvywbssce_0de2fe96-aede-41b0-9f34-1d273f895a69   4m7s
apiserver-zbenbsdlw5akl66fz2mryo2cjm   apiserver-zbenbsdlw5akl66fz2mryo2cjm_441d820a-88d0-4877-a946-1ddf818e1c5a   4m2s
kube-controller-manager                k8s-master-3_0be24ea8-c6b4-4567-a973-1fd02b891351                           4m5s
kube-scheduler                         k8s-master-3_a514f041-cb41-4e71-bc6e-3169abbb7a36                           4m4s
```

## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#checking-api-access) API to determine authorization.

The commands in this section will affect the entire cluster and only need to be run on the k8s-master-1 machine.

```bash
ssh root@k8s-master-1
```

Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```bash
kubectl apply -f kube-apiserver-to-kubelet.yaml \
  --kubeconfig admin.kubeconfig
```

### Verification

At this point the Kubernetes control plane is up and running. Run the following commands from the `jumpbox` machine to verify it's working:

Make a HTTP request for the Kubernetes version info:

```bash
curl --cacert /var/lib/kubernetes/ca.crt \
  https://server.kubernetes.local:6443/version
```

```text
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  "gitCommit": "32cc146f75aad04beaaa245a7157eb35063a9f99",
  "gitTreeState": "clean",
  "buildDate": "2025-03-11T19:52:21Z",
  "goVersion": "go1.23.6",
  "compiler": "gc",
  "platform": "linux/arm64"
}
```

Next: [Bootstrapping the Kubernetes Worker Nodes](10-bootstrapping-kubernetes-masters-and-workers.md)
