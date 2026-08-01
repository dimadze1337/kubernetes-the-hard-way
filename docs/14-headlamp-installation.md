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

#helm install ingress-nginx ingress-nginx-*.tgz \
#  --namespace ingress-nginx \
#  --create-namespace

helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=32080 \
  --set controller.service.nodePorts.https=32443
```

# Verify

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

```text
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7558b45cf6-kbnsz   1/1     Running   0          17s

NAME                                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.0.0.19    <none>        80:32080/TCP,443:32443/TCP   37s
ingress-nginx-controller-admission   ClusterIP   10.0.0.144   <none>        443/TCP                      37s

```


# Install Headlamp

```bash
#helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
#helm repo update

#helm install headlamp headlamp/headlamp \
  -n headlamp \
#  --create-namespace

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/headlamp/main/kubernetes-headlamp.yaml

kubectl get svc -n kube-system | grep "headlamp"
kubectl get po -n kube-system | grep "headlamp"
```

```text
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
headlamp   ClusterIP   10.0.0.200   <none>        80/TCP    77s

NAME                      READY   STATUS    RESTARTS   AGE
headlamp-747b5f4d5-wsvhx   1/1     Running   0         56s
```

# Exposing Headlamp with an ingress server

```bash
cat > headlamp-ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: headlamp
  namespace: kube-system
spec:
  ingressClassName: nginx
  rules:
  - host: headlamp.kubernetes.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: headlamp
            port:
              number: 80
EOF
```

```bash
kubectl apply -f headlamp-ingress.yaml
```

# Check Headlamp availability
```bash
curl http://headlamp.kubernetes.local/
```

Or open in browser `http://headlamp.kubernetes.local/`

# Accessing Headlamp

Going with the manual from https://headlamp.dev/docs/latest/installation/in-cluster/#using-simple-yaml

Once Headlamp is up and running, be sure to enable access to it either by creating a service account or by setting up OIDC.

```bash
cat > headlamp-crb.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: headlamp-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: headlamp-admin
  namespace: kube-system
EOF

cat > headlamp-sa.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: headlamp-admin
  namespace: kube-system
EOF


kubectl apply -f headlamp-sa.yaml
kubectl apply -f headlamp-crb.yaml
kubectl create token headlamp-admin -n kube-system
```