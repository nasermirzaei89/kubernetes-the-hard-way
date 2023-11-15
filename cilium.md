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
  --version 1.14.4
```

## Verify

```shell
kubectl get nodes
```

Now nodes are ready.

To check Cilium status, install Cilium CLI
from https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli

```shell
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

Then:

```shell
cilium status --wait
```

To test Cilium:

```shell
cilium connectivity test
```

> Note: Your cluster should have at least 2 worker node to pass this test.
