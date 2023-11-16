# CoreDNS

```shell
helm upgrade --install coredns --repo https://coredns.github.io/helm coredns \
  --namespace=kube-system \
  --atomic \
  --set service.clusterIP=172.16.0.10 \
  --version 1.28.1
```
