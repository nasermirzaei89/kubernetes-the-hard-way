# Metrics Server

To enable metrics, you need to install metrics-server.

## Install

```shell
helm upgrade --install metrics-server --repo https://kubernetes-sigs.github.io/metrics-server/ metrics-server \
  --namespace metrics-server --create-namespace \
  --atomic \
  --version v3.11.0
```

## Verify

for Test:

```shell
kubectl top nodes
```
