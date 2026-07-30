# Install Helm on `jumpbox` machine
```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo update
```

# Taint master nodes

```bash
kubectl taint nodes k8s-master-1 node-role.kubernetes.io/control-plane=:NoSchedule
kubectl taint nodes k8s-master-2 node-role.kubernetes.io/control-plane=:NoSchedule
kubectl taint nodes k8s-master-3 node-role.kubernetes.io/control-plane=:NoSchedule
```

# Install CoreDNS

```bash
cat > coredns.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local 10.200.0.0/16 {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 1.1.1.1
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
spec:
  clusterIP: 10.0.0.10
  selector:
    k8s-app: kube-dns
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      serviceAccountName: coredns
      containers:
      - name: coredns
        image: registry.k8s.io/coredns/coredns:v1.12.1
        args:
        - -conf
        - /etc/coredns/Corefile
        volumeMounts:
        - name: config
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          protocol: UDP
        - containerPort: 53
          protocol: TCP
      volumes:
      - name: config
        configMap:
          name: coredns
EOF
```

```bash
cat > coredns-rbac.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:coredns
rules:
- apiGroups: [""]
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch

- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
EOF
```

# Create ClusterRoleBinding for kube-proxy

```bash
cat > kube-proxy-clusterrolebinding.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-proxy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node-proxier
subjects:
- kind: User
  name: system:kube-proxy
  apiGroup: rbac.authorization.k8s.io
EOF
```

# Apply yamls

```bash
kubectl apply -f kube-proxy-clusterrolebinding.yaml
kubectl apply -f coredns-rbac.yaml
kubectl apply -f coredns.yaml

kubectl get svc -n kube-system
kubectl get pods -n kube-system
```

```text
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.0.0.10    <none>        53/UDP,53/TCP   11s


NAME                       READY   STATUS              RESTARTS   AGE
coredns-784c699c6b-cfncj   1/1     Running             0          89s
coredns-784c699c6b-6f8rq   1/1     Running             0          89s

```

# Check rights

```bash
kubectl auth can-i list services --as=system:kube-proxy
kubectl auth can-i watch endpointslices --as=system:kube-proxy
kubectl auth can-i watch nodes --as=system:kube-proxy
```

# On worker nodes
```bash
for host in k8s-worker-1 k8s-worker-2 k8s-worker-3; do
  ssh root@${host} '
    systemctl restart kube-proxy
  '
done
```

# Install ingress-nginx

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx-*.tgz \
  --namespace ingress-nginx \
  --create-namespace
  
kubectl get pods -n ingress-nginx
```

```text
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7558b45cf6-hxcjm   1/1     Running   0          81s
```

# Install Headlamp

```bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo update

helm install headlamp headlamp/headlamp \
  -n headlamp \
  --create-namespace
  
kubectl get svc -n headlamp
kubectl get pods -n headlamp
```

```text
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
headlamp   ClusterIP   10.0.0.196   <none>        80/TCP    69s

NAME                      READY   STATUS    RESTARTS   AGE
headlamp-bdbd8bc4-l7z7f   1/1     Running   0          83s
```

