# Metallb

## Install HCloud Floating IP Controller

https://github.com/cbeneke/hcloud-fip-controller/blob/v0.4.1/docs/deploy.md

export `HCLOUD_API_TOKEN` first.

Then:

```shell
kubectl create namespace fip-controller

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: fip-controller-config
  namespace: fip-controller
data:
  config.json: |
    {
      "floating_ips_label_selector": "cluster=applicaset-ash1",
      "log_level": "debug"
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: fip-controller-secrets
  namespace: fip-controller
stringData:
  HCLOUD_API_TOKEN: "${HCLOUD_API_TOKEN}"
EOF

kubectl apply -f https://raw.githubusercontent.com/cbeneke/hcloud-fip-controller/v0.4.1/deploy/rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/cbeneke/hcloud-fip-controller/v0.4.1/deploy/deployment.yaml
```

## Install Metallb

```shell
helm upgrade --install metallb --repo https://metallb.github.io/metallb metallb \
  --namespace metallb-system --create-namespace \
  --atomic \
  --version 0.13.12
```

### Add floating ip to Metallb

```shell
FLOATING_IP_NAME=floating-ipv4-$(head -c 16 /dev/urandom | base64 | tr -d '+/' | tr -dc 'a-z0-9' | head -c 10)
CLUSTER_NAME=cluster1
hcloud floating-ip create --name ${FLOATING_IP_NAME} --type ipv4 --home-location ash  --label cluster=${CLUSTER_NAME}
```

### Add it to worker nodes

To list floating IPs:

```shell
hcloud floating-ip list --selector cluster=${CLUSTER_NAME}
```

On all worker nodes

```shell
ip addr add <IP> dev eth0
```

Replace `<IP>` with desired IP address.

```shell
cat <<EOF | kubectl apply -f-
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool1
  namespace: metallb-system
spec:
  addresses:
    - <IP>/32
EOF
```

Replace `<IP>` with desired ip address.

Now you can use these IPs for services with type `LoadBalancer`.
