# Bootstrapping the Kubernetes Control Planes

## Generate and Sign the Kubernetes Admin Client Certificate

```shell
openssl genrsa -out kubernetes-admin.key 2048

openssl req -new -key kubernetes-admin.key -subj "/CN=kubernetes-admin/O=system:masters" -out kubernetes-admin.csr
openssl x509 -req -in kubernetes-admin.csr -CA kubernetes-ca.crt -CAkey kubernetes-ca.key \
  -sha256 -CAcreateserial -days 730 \
  -out kubernetes-admin.crt
```

```shell
kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=kubernetes-ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kubernetes-admin.kubeconfig

kubectl config set-credentials kubernetes-admin \
  --client-certificate=kubernetes-admin.crt \
  --client-key=kubernetes-admin.key \
  --embed-certs=true \
  --kubeconfig=kubernetes-admin.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=kubernetes-admin \
  --kubeconfig=kubernetes-admin.kubeconfig

kubectl config use-context default --kubeconfig=kubernetes-admin.kubeconfig
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector control-plane=true -o noheader -o columns=name); do
  scp kubernetes-admin.kubeconfig root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/
done
```

## Generate and Sign the Controller Manager Client Certificate

```shell
openssl genrsa -out kube-controller-manager.key 2048

openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager/O=system:kube-controller-manager" -out kube-controller-manager.csr
openssl x509 -req -in kube-controller-manager.csr -CA kubernetes-ca.crt -CAkey kubernetes-ca.key \
  -sha256 -CAcreateserial -days 730 \
  -out kube-controller-manager.crt
```

```shell
kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=kubernetes-ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.crt \
  --client-key=kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector control-plane=true -o noheader -o columns=name); do
  scp kube-controller-manager.kubeconfig root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/
done
```

## Generate and Sign the Scheduler Client Certificate

```shell
openssl genrsa -out kube-scheduler.key 2048

openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler/O=system:kube-scheduler" -out kube-scheduler.csr
openssl x509 -req -in kube-scheduler.csr -CA kubernetes-ca.crt -CAkey kubernetes-ca.key \
  -sha256 -CAcreateserial -days 730 \
  -out kube-scheduler.crt
```

```shell
kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=kubernetes-ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.crt \
  --client-key=kube-scheduler.key \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector control-plane=true -o noheader -o columns=name); do
  scp kube-scheduler.kubeconfig root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/
done
```

## Generate and Sign the Kubernetes API Server Certificate

Create a template file for config:

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector load-balancer=true -o noheader -o columns=name); do
  LOAD_BALANCER_INTERNAL_IP=$(hcloud server describe ${NODE_NAME} -o format='{{ (index .PrivateNet 0).IP}}')
done
cat <<EOF | tee kubernetes.conf
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = kubernetes

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1= kubernetes
DNS.2= kubernetes.default
DNS.3= kubernetes.default.svc
DNS.4= kubernetes.default.svc.cluster
DNS.5= kubernetes.default.svc.cluster.local
IP.1 = 172.16.0.1
IP.2 = ${LOAD_BALANCER_INTERNAL_IP}
IP.3 = 127.0.0.1

[ v3_ext ]
subjectAltName=@alt_names
EOF
```

We use `172.16.0.0/12` for Service ip range. So the first one will be `kubernetes` service.

```shell
openssl genrsa -out kubernetes.key 2048

openssl req -new -key kubernetes.key -config kubernetes.conf -out kubernetes.csr
openssl x509 -req -in kubernetes.csr -CA kubernetes-ca.crt -CAkey kubernetes-ca.key \
  -sha256 -CAcreateserial -days 730 -extensions v3_ext -extfile kubernetes.conf \
  -out kubernetes.crt
```


```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector control-plane=true -o noheader -o columns=name); do
  scp \
  kubernetes-ca.crt kubernetes-ca.key \
  kubernetes.crt kubernetes.key \
  root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/
done
```

## Generate and Sign the Service Account Key Pair

```shell
openssl genrsa -out service-accounts.key 2048

openssl req -new -key service-accounts.key -subj "/CN=service-accounts" -out service-accounts.csr
openssl x509 -req -in service-accounts.csr -CA kubernetes-ca.crt -CAkey kubernetes-ca.key \
  -sha256 -CAcreateserial -days 730 \
  -out service-accounts.crt
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector control-plane=true -o noheader -o columns=name); do
  scp service-accounts.crt service-accounts.key root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/
done
```

## Generating the Data Encryption Config and Key

Generate an encryption key:

```shell
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

Create the `encryption-config.yaml` encryption config file:

```shell
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector control-plane=true -o noheader -o columns=name); do
  scp encryption-config.yaml root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/
done
```


### Create the Front-end Proxy Keys

```shell
openssl genrsa -out front-proxy-ca.key 2048
openssl req -x509 -new -nodes -key front-proxy-ca.key -subj "/CN=kubernetes-front-proxy-ca" -days 5475 -out front-proxy-ca.crt
```

```shell
openssl genrsa -out front-proxy-client.key 2048

openssl req -new -key front-proxy-client.key -subj "/CN=front-proxy-client" -out front-proxy-client.csr
openssl x509 -req -in front-proxy-client.csr -CA front-proxy-ca.crt -CAkey front-proxy-ca.key \
  -sha256 -CAcreateserial -days 730 \
  -out front-proxy-client.crt
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector control-plane=true -o noheader -o columns=name); do
  scp front-proxy-client.crt front-proxy-client.key front-proxy-ca.crt root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/
done
```



## Install

### SSH to Node

On all control plane nodes:

```shell
ssh root@$(hcloud server describe <NODE_NAME> -o format='{{.PublicNet.IPv4.IP}}')
```

Replace `<NODE_NAME>` with real node names.

### Download binaries

```shell
mkdir -p /etc/kubernetes/config
```

```shell
KUBERNETES_VERSION=v1.28.3
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kubectl"
```

```shell
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

```shell
mkdir -p /var/lib/kubernetes/

mv kubernetes-ca.crt kubernetes-ca.key \
  kubernetes.crt kubernetes.key \
  service-accounts.crt service-accounts.key \
  front-proxy-client.crt front-proxy-client.key front-proxy-ca.crt \
  encryption-config.yaml /var/lib/kubernetes/
```

### Create `kube-apiserver` Service

```shell
cat <<EOF | tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --authorization-mode=Node,RBAC \\
  --client-ca-file=/var/lib/kubernetes/kubernetes-ca.crt \\
  --etcd-servers=https://${INTERNAL_IP}:2379 \\
  --etcd-cafile=/etc/etcd/ca.crt \\
  --etcd-certfile=/etc/etcd/server.crt \\
  --etcd-keyfile=/etc/etcd/server.key \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/kubernetes-ca.crt \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.crt \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes.key \\
  --service-account-issuer=https://${INTERNAL_IP}:6443 \\
  --service-account-key-file=/var/lib/kubernetes/service-accounts.crt \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-accounts.key \\
  --service-cluster-ip-range=172.16.0.0/12 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.crt \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes.key \\
  --proxy-client-cert-file=/var/lib/kubernetes/front-proxy-client.crt \\
  --proxy-client-key-file=/var/lib/kubernetes/front-proxy-client.key \\
  --requestheader-allowed-names=front-proxy-client \\
  --requestheader-client-ca-file=/var/lib/kubernetes/front-proxy-ca.crt \\
  --requestheader-extra-headers-prefix=X-Remote-Extra- \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Create `kube-controller-manager` Service

```shell
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

```shell
cat <<EOF | tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=${CLUSTER_NAME} \\
  --cluster-signing-cert-file=/var/lib/kubernetes/kubernetes-ca.crt \\
  --cluster-signing-key-file=/var/lib/kubernetes/kubernetes-ca.key \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --root-ca-file=/var/lib/kubernetes/kubernetes-ca.crt \\
  --service-account-private-key-file=/var/lib/kubernetes/service-accounts.key \\
  --service-cluster-ip-range=172.16.0.0/12 \\
  --use-service-account-credentials=true

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```shell
mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

```shell
cat <<EOF | tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
EOF
```

```shell
cat <<EOF | tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --authentication-kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --authorization-kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```shell
systemctl daemon-reload
systemctl enable kube-apiserver kube-controller-manager kube-scheduler
systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

## Connect kubectl

```shell
mkdir -p .kube
mv kubernetes-admin.kubeconfig .kube/config
```

## Enable Shell Autocompletion

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion

```shell
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

If you have an alias for kubectl, you can extend shell completion to work with that alias:

```shell
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
```

## RBAC for Kubelet Authorization

Run:

```shell
kubectl get clusterrole system:kubelet-api-admin -o yaml
```

to make sure it exists. 

```shell
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubelet-api-admin
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```
