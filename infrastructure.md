# Provisioning Infrastructure

Make sure you configured Hetzner CLI.

## Recommended specs

For a High Available Kubernetes cluster we need:

* 3 etcd nodes
* 3 control planes
* 1 worker node
* 1 load balancer

We create a virtual private network and some servers.

For testing purposes you can use 1 light server (cpx11) for all these components, but for production we recommend:

* For control planes, minimum 2 core cpu, maximum 4-8 core cpu.
* For workers, minimum 2-4 core cpu, maximum 16-32 core cpu.
* For control planes, minimum 2-4 GB ram, maximum 8-16 GB ram.
* For workers, minimum 4-8 GB ram, maximum 64-128 GB ram.

For cluster storage, it's recommended to attach separate disks for using dedicated,
but don't forget containers download images on nodes and servers needs significant space to store them.

## Creating resources

We label servers with their functionality here and in next steps we find them by labels.

### Create VPC

you can change variable names based on your preferences.

```shell
NETWORK_NAME=network1
hcloud network create --name ${NETWORK_NAME} --ip-range 10.0.0.0/8
```

```shell
hcloud network add-subnet ${NETWORK_NAME} --network-zone us-east --type server --ip-range 10.1.0.0/16
```

I decided to use location `ash` which is in network zone `us-east` region. you can use other locations.
Get the list of available locations by:

```shell
hcloud location list
```

### Create an SSH Key

```shell
SSH_KEY_NAME=ssh-key-1
hcloud ssh-key create --name ${SSH_KEY_NAME} --public-key-from-file ~/.ssh/id_ed25519.pub
```

Use your own public key path to upload it.

### Create Servers

For testing create a server with all labels:

```shell
NODE_NAME=node-$(head -c 16 /dev/urandom | base64 | tr -d '+/' | tr -dc 'a-z0-9' | head -c 10)
CLUSTER_NAME=cluster1
hcloud server create \
  --name ${NODE_NAME} \
  --network ${NETWORK_NAME} \
  --image ubuntu-22.04 \
  --type cpx11 \
  --ssh-key ${SSH_KEY_NAME} \
  --label cluster=${CLUSTER_NAME} \
  --label etcd=true \
  --label control-plane=true \
  --label worker=true \
  --label load-balancer=true \
  --location ash
```

for production create separate servers with these labels:

* 3 servers with labels `etcd=true` and `control-plane=true`
* 1 server with label `load-balancer=true`
* at least 1 server with label `worker=true`

All servers should have cluster label to separate them from other clusters if you create.
