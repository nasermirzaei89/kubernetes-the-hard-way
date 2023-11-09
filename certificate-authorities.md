# Provisioning Certificate Authorities

## Useful links

* https://kubernetes.io/docs/tasks/administer-cluster/certificates/
* https://kubernetes.io/docs/setup/best-practices/certificates/

## Create CAs

According to kubernetes docs, we need 3 certificate authorities:

* `kubernetes-ca` as Kubernetes general CA
* `etcd-ca` for all etcd-related functions
* `kubernetes-front-proxy-ca` for
  the [front-end proxy](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)

### Kubernetes CA

```shell
openssl genrsa -out kubernetes-ca.key 2048
openssl req -x509 -new -nodes -key kubernetes-ca.key -subj "/CN=kubernetes-ca" -days 5475 -out kubernetes-ca.crt
```

5475 days is 15 years, You can use more or less days.

### etcd CA

```shell
openssl genrsa -out etcd-ca.key 2048
openssl req -x509 -new -nodes -key etcd-ca.key -subj "/CN=etcd-ca" -days 5475 -out etcd-ca.crt
```

### Front Proxy CA

```shell
openssl genrsa -out kubernetes-front-proxy-ca.key 2048
openssl req -x509 -new -nodes -key kubernetes-front-proxy-ca.key -subj "/CN=kubernetes-front-proxy-ca" -days 5475 -out kubernetes-front-proxy-ca.crt
```

Later we can sign and verify all created certificates by these CAs.
