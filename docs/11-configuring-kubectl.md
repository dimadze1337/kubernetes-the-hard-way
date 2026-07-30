# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the `jumpbox` machine.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to.

You should be able to ping `lb.kubernetes.local` (HaProxy balancer server) based on the `/etc/hosts` DNS entry from a previous lab.

```bash
curl --cacert ca.crt \
  https://lb.kubernetes.local:6443/version
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

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://lb.kubernetes.local:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```
The results of running the command above should create a kubeconfig file in the default location `~/.kube/config` used by the  `kubectl` commandline tool. This also means you can run the `kubectl` command without specifying a config.

## Add kubeconfig on masters

```bash
for host in k8s-master-1 k8s-master-2 k8s-master-3; do
  ssh root@${host} '
    mkdir -p ~/.kube
    mv ~/admin.kubeconfig ~/.kube/config
    chmod 600 ~/.kube/config
  '
done
```

## Verification

Check the version of the remote Kubernetes cluster:

```bash
kubectl version
```

```text
Client Version: v1.32.3
Kustomize Version: v5.5.0
Server Version: v1.32.3
```

List the nodes in the remote Kubernetes cluster:

```bash
kubectl get nodes
```

```
NAME     STATUS   ROLES    AGE    VERSION
k8s-master-1   Ready    <none>   2m32s   v1.32.3
k8s-master-2   Ready    <none>   2m35s   v1.32.3
k8s-master-3   Ready    <none>   2m30s   v1.32.3
k8s-worker-1   Ready    <none>   2m34s   v1.32.3
k8s-worker-2   Ready    <none>   2m29s   v1.32.3
k8s-worker-3   Ready    <none>   2m28s   v1.32.3

```

Next: [Provisioning Pod Network Routes](12-pod-network-routes.md)
