# Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```bash
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```bash
ssh root@k8s-master-1 \
    'etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C'
```

```text
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 51 42 17 fd ce 85 fe  |:v1:key1:QB.....|
00000050  8c 2f 3d 92 92 41 66 83  c5 cc 8d 6f f5 82 93 26  |./=..Af....o...&|
00000060  aa 47 6d 85 fc 16 d5 1b  9d ee ed c1 5e ac ce ab  |.Gm.........^...|
00000070  d3 03 59 c3 79 a7 5a 7c  15 f3 47 0d da b0 97 7d  |..Y.y.Z|..G....}|
00000080  fa 7d 5d a3 da 28 85 a0  f7 e8 e5 b1 2b ac 79 90  |.}]..(......+.y.|
00000090  2d 61 16 25 76 2b 25 e9  16 89 37 cb ef dc 4b 60  |-a.%v+%...7...K`|
000000a0  49 8f 57 9a ba d0 cd d9  1b f9 67 71 ae 2b e6 bc  |I.W.......gq.+..|
000000b0  f2 1b d3 c6 ad 47 bd 44  19 c0 ed f0 36 7f a7 e3  |.....G.D....6...|
000000c0  5c aa 12 8f e2 9f 06 a4  19 31 6e 34 93 7a 7e 6a  |\........1n4.z~j|
000000d0  29 01 da 66 ef 24 09 0e  6e 9f 65 a6 c1 21 08 19  |)..f.$..n.e..!..|
000000e0  da 6f 8d df 5d 32 b6 8c  e4 a5 77 c4 be b9 ce 1b  |.o..]2....w.....|
000000f0  8a 00 e8 5b f7 26 8e 94  87 ad ae ec e7 b0 55 8b  |...[.&........U.|
00000100  90 81 98 b6 93 9c 7f ba  1a e4 b6 6b 11 ef bc 4a  |...........k...J|
00000110  77 25 78 f0 08 e7 98 14  42 c5 d4 a1 e8 2e 60 aa  |w%x.....B.....`.|
00000120  a1 bd 55 d7 c4 f3 5a d4  24 4b 6f df 22 1a 34 a2  |..U...Z.$Ko.".4.|
00000130  8a df 01 f7 39 75 47 8f  92 2e 5f 9a 1b b6 46 7e  |....9uG..._...F~|
00000140  f8 e5 ad e8 0d 9f ef 82  1a d5 a6 50 82 3e e7 65  |...........P.>.e|
00000150  72 25 91 86 6d 6f af 27  b6 0a                    |r%..mo.'..|
0000015a

```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```bash
kubectl create deployment nginx \
  --image=nginx:latest
```

List the pod created by the `nginx` deployment:

```bash
kubectl get pods -l app=nginx
```

```bash
NAME                     READY   STATUS    RESTARTS   AGE
nginx-54c98b4f84-2qz5q   1/1     Running   0          102s
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```bash
POD_NAME=$(kubectl get pods -l app=nginx \
  -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```bash
kubectl port-forward $POD_NAME 8080:80
```

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```bash
curl --head http://127.0.0.1:8080
```

```text
HTTP/1.1 200 OK
Server: nginx/1.27.4
Date: Sun, 06 Apr 2025 17:17:12 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 05 Feb 2025 11:06:32 GMT
Connection: keep-alive
ETag: "67a34638-267"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```bash
kubectl logs $POD_NAME
```

```text
...
127.0.0.1 - - [06/Apr/2025:17:17:12 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.88.1" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```bash
kubectl exec -ti $POD_NAME -- nginx -v
```

```text
nginx version: nginx/1.27.4
```

## Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```bash
kubectl expose deployment nginx \
  --port 80 --type NodePort
```

> The LoadBalancer service type can not be used because your cluster is not configured with [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider). Setting up cloud provider integration is out of scope for this tutorial.

Retrieve the node port assigned to the `nginx` service:

```bash
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Retrieve the hostname of the node running the `nginx` pod:

```bash
NODE_NAME=$(kubectl get pods \
  -l app=nginx \
  -o jsonpath="{.items[0].spec.nodeName}")
```

Make an HTTP request using the IP address and the `nginx` node port:

```bash
curl -I http://${NODE_NAME}:${NODE_PORT}
```

```text
Server: nginx/1.27.4
Date: Sun, 06 Apr 2025 17:18:36 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 05 Feb 2025 11:06:32 GMT
Connection: keep-alive
ETag: "67a34638-267"
Accept-Ranges: bytes
```

Next: [Headlamp installation](14-headlamp-installation.md)
