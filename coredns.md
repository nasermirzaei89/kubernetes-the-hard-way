# CoreDNS

```shell
helm upgrade --install coredns --repo https://coredns.github.io/helm coredns \
  --namespace=kube-system \
  --atomic
```