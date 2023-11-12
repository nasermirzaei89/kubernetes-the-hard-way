# NGINX Ingress Controller

## Dependencies

[Install Metallb](./metallb.md)

## Install NGINX Ingress Controller

```shell
helm upgrade --install ingress-nginx --repo https://kubernetes.github.io/ingress-nginx ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --atomic \
  --version 4.8.3
```

Get ingress ip by:

```shell
kubectl get service -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Add an `A` record with name `*.ash1` and ingres ip to your custom domain. For example `example.page`.

## Install Cert Manager

```shell
helm upgrade --install cert-manager --repo https://charts.jetstack.io cert-manager \
  --namespace cert-manager --create-namespace \
  --atomic \
  --set installCRDs=true \
  --version v1.13.2
```

### Let’s Encrypt Issuer

https://cert-manager.io/docs/tutorials/acme/nginx-ingress/#step-6---configure-a-lets-encrypt-issuer

#### Staging

```shell
kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/staging-issuer.yaml
```

- Change `Issuer` to `ClusterIssuer`
- Remove `namespace`
- Remove `email`

#### Production

```shell
kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/production-issuer.yaml
```

- Change `Issuer` to `ClusterIssuer`
- Remove `namespace`
- Remove `email`
