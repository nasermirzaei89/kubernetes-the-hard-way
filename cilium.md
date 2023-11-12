# Cilium

## Install Helm

https://helm.sh/docs/intro/install/#from-apt-debianubuntu

```shell
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Install Chart

```shell
helm upgrade --install cilium --repo https://helm.cilium.io/ cilium \
  --namespace kube-system \
  --atomic \
  --set operator.replicas=1 \
  --version 1.14.3
```

## Verify

```shell
kubectl get nodes
```

Now nodes are ready.
