# Add Worker Nodes

## Generate and Sign the Kubelet client certificate

FOR EACH Worker:::

Create a template file for config:

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector worker=true -o noheader -o columns=name); do
  INTERNAL_IP=$(hcloud server describe ${NODE_NAME} -o format='{{ (index .PrivateNet 0).IP}}')
  cat <<EOF | tee ${NODE_NAME}-kubelet.conf
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = system:node:${NODE_NAME}
O = system:nodes

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1= ${NODE_NAME}
IP.1 = ${INTERNAL_IP}

[ v3_ext ]
subjectAltName=@alt_names
EOF
done
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector worker=true -o noheader -o columns=name); do
  openssl genrsa -out ${NODE_NAME}-kubelet.key 2048
  
  openssl req -new -key ${NODE_NAME}-kubelet.key \
    -config ${NODE_NAME}-kubelet.conf \
    -out ${NODE_NAME}-kubelet.csr
  openssl x509 -req -in ${NODE_NAME}-kubelet.csr -CA kubernetes-ca.crt -CAkey kubernetes-ca.key \
    -sha256 -CAcreateserial -days 730 -extensions v3_ext -extfile ${NODE_NAME}-kubelet.conf \
    -out ${NODE_NAME}-kubelet.crt
done
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector worker=true -o noheader -o columns=name); do
  kubectl config set-cluster ${CLUSTER_NAME} \
    --certificate-authority=kubernetes-ca.crt \
    --embed-certs=true \
    --server=https://${LOAD_BALANCER_INTERNAL_IP}:6443 \
    --kubeconfig=${NODE_NAME}-kubelet.kubeconfig

  kubectl config set-credentials system:node:${NODE_NAME} \
    --client-certificate=${NODE_NAME}-kubelet.crt \
    --client-key=${NODE_NAME}-kubelet.key \
    --embed-certs=true \
    --kubeconfig=${NODE_NAME}-kubelet.kubeconfig

  kubectl config set-context default \
    --cluster=${CLUSTER_NAME} \
    --user=system:node:${NODE_NAME} \
    --kubeconfig=${NODE_NAME}-kubelet.kubeconfig

  kubectl config use-context default --kubeconfig=${NODE_NAME}-kubelet.kubeconfig
done
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector worker=true -o noheader -o columns=name); do
  scp ${NODE_NAME}-kubelet.kubeconfig root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/kubelet.kubeconfig
  scp ${NODE_NAME}-kubelet.crt root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/kubelet.crt
  scp ${NODE_NAME}-kubelet.key root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/kubelet.key
done
```


## Generate and Sign the Kube Proxy client certificate

```shell
openssl genrsa -out kube-proxy.key 2048

openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy/O=system:node-proxier" -out kube-proxy.csr
openssl x509 -req -in kube-proxy.csr -CA kubernetes-ca.crt -CAkey kubernetes-ca.key \
  -sha256 -CAcreateserial -days 730 \
  -out kube-proxy.crt
```

```shell
kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=kubernetes-ca.crt \
  --embed-certs=true \
  --server=https://${LOAD_BALANCER_INTERNAL_IP}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.crt \
  --client-key=kube-proxy.key \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

```shell
for NODE_NAME in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector worker=true -o noheader -o columns=name); do
  scp kube-proxy.kubeconfig kubernetes-ca.crt root@$(hcloud server describe ${NODE_NAME} -o format='{{.PublicNet.IPv4.IP}}'):~/
done
```

## SSH to Node

On all worker nodes:

```shell
ssh root@$(hcloud server describe <NODE_NAME> -o format='{{.PublicNet.IPv4.IP}}')
```

## Turn off SWAP

```shell
swapoff -a
```

## Install Containerd

[Install Containerd](./containerd.md)

## Download binaries

```shell
KUBERNETES_VERSION=v1.28.3
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kubelet
```

```shell
chmod +x kube-proxy kubelet
mv kube-proxy kubelet /usr/local/bin/
```

```shell
mkdir -p /var/lib/kubelet/
mkdir -p /var/lib/kubernetes/
mv kubelet.key kubelet.crt /var/lib/kubelet/
mv kubelet.kubeconfig /var/lib/kubelet/kubeconfig
mv kubernetes-ca.crt /var/lib/kubernetes/
```


```shell
cat <<EOF | tee /var/lib/kubelet/kubelet-config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/kubernetes-ca.crt"
authorization:
  mode: Webhook
cgroupDriver: systemd
clusterDNS:
  - "172.16.0.10"
clusterDomain: "cluster.local"
podCIDR: "10.200.0.0/16"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/kubelet.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/kubelet.key"
EOF
```

```shell
cat <<EOF | tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --node-ip=${INTERNAL_IP} \\
  --kubeconfig=/var/lib/kubelet/kubeconfig
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```shell
mkdir -p /var/lib/kube-proxy/
mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

```shell
cat <<EOF | tee /var/lib/kube-proxy/kube-proxy-config.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "ipvs"
bindAddress: 0.0.0.0
clusterCIDR: "10.200.0.0/16"
EOF
```

if you use mode "ipvs", you need to install `ipset` and `conntrack`:

```shell
apt install -y ipset conntrack
```

```shell
cat <<EOF | tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```shell
systemctl daemon-reload
systemctl enable kubelet kube-proxy
systemctl start kubelet kube-proxy
```

## Verify

Run:

```shell
kubectl get nodes
```

Now you see nodes are not ready.
To make them ready you need a CNI.
