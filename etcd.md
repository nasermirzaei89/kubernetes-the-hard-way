# etcd Cluster

## Generate and Sign etcd Member Certificates

Create a template file for config:

```shell
cat <<EOF | tee etcd-server.conf.tpl
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = <NODE_NAME>

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = <NODE_NAME>
DNS.2 = localhost
IP.1 = 127.0.0.1
IP.2 = <NODE_INTERNAL_IP>

[ v3_ext ]
subjectAltName=@alt_names
EOF
```

We need 2 certificates for each etcd node. Server certificate for authenticating client requests, and Peer certificate to communicate with other nodes.

Although it’s not mandatory, using separate certificates for different purposes adds an extra layer of security and minimizes the risk of compromise in a distributed system like etcd.

For each member (Server Cert):

```shell
for NODE in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector etcd=true -o noheader -o columns=name); do
  openssl genrsa -out ${NODE}-etcd-server.key 2048

  INTERNAL_IP=$(hcloud server describe ${NODE} -o format='{{ (index .PrivateNet 0).IP}}')

  cat etcd-server.conf.tpl | sed -e "s/\<NODE_NAME>/${NODE}/" -e "s/\<NODE_INTERNAL_IP>/${INTERNAL_IP}/" > ${NODE}-etcd-server.conf 

  openssl req -new -key ${NODE}-etcd-server.key \
    -config ${NODE}-etcd-server.conf \
    -out ${NODE}-etcd-server.csr
  openssl x509 -req -in ${NODE}-etcd-server.csr -CA etcd-ca.crt -CAkey etcd-ca.key \
    -sha256 -CAcreateserial -days 730 -extensions v3_ext -extfile ${NODE}-etcd-server.conf \
    -out ${NODE}-etcd-server.crt
done
```

For each member (Peer Cert):

```shell
for NODE in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector etcd=true -o noheader -o columns=name); do
  openssl genrsa -out ${NODE}-etcd-peer.key 2048

  EXTERNAL_IP=$(hcloud server describe ${NODE} -o format='{{.PublicNet.IPv4.IP}}')
  INTERNAL_IP=$(hcloud server describe ${NODE} -o format='{{ (index .PrivateNet 0).IP}}')

  cat etcd-server.conf.tpl | sed -e "s/\<NODE_NAME>/${NODE}/" -e "s/\<NODE_INTERNAL_IP>/${INTERNAL_IP}/" -e "s/\<NODE_EXTERNAL_IP>/${EXTERNAL_IP}/" > ${NODE}-etcd-peer.conf 

  openssl req -new -key ${NODE}-etcd-peer.key \
    -config ${NODE}-etcd-peer.conf \
    -out ${NODE}-etcd-peer.csr
  openssl x509 -req -in ${NODE}-etcd-peer.csr -CA etcd-ca.crt -CAkey etcd-ca.key \
    -sha256 -CAcreateserial -days 730 -extensions v3_ext -extfile ${NODE}-etcd-peer.conf \
    -out ${NODE}-etcd-peer.crt
done
```

Copy Certificates to servers

```shell
for NODE in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector etcd=true -o noheader -o columns=name); do
  scp etcd-ca.crt root@$(hcloud server describe ${NODE} -o format='{{.PublicNet.IPv4.IP}}'):~/
  scp ${NODE}-etcd-server.key root@$(hcloud server describe ${NODE} -o format='{{.PublicNet.IPv4.IP}}'):~/etcd-server.key
  scp ${NODE}-etcd-server.crt root@$(hcloud server describe ${NODE} -o format='{{.PublicNet.IPv4.IP}}'):~/etcd-server.crt
  scp ${NODE}-etcd-peer.key root@$(hcloud server describe ${NODE} -o format='{{.PublicNet.IPv4.IP}}'):~/etcd-peer.key
  scp ${NODE}-etcd-peer.crt root@$(hcloud server describe ${NODE} -o format='{{.PublicNet.IPv4.IP}}'):~/etcd-peer.crt
done
```

## Install

### SSH to Node

On all etcd nodes:

```shell
ssh root@$(hcloud server describe <NODE_NAME> -o format='{{.PublicNet.IPv4.IP}}')
```

Replace `<NODE_NAME>` with real node names.

### Install etcd

```shell
apt update
apt install -y etcd
```

In this article etcd 3.5.10 is released. Future releases may have a different configuration.

### Configure

Now etcd is installed and running in single mode on each node.
So, we need to update the configurations.

I create a subdirectory for `etcd` in its favorite parent directory.
Move and rename all certificates there.
Then, change the directory owner to the use etcd, which has been created on installing etcd.

```shell
mkdir -p /etc/etcd
mv ~/etcd-ca.crt /etc/etcd/ca.crt
mv ~/etcd-server.key /etc/etcd/server.key
mv ~/etcd-server.crt /etc/etcd/server.crt
mv ~/etcd-peer.key /etc/etcd/peer.key
mv ~/etcd-peer.crt /etc/etcd/peer.crt
chown -R etcd /etc/etcd
```

Now we need to update etcd config values to make them a cluster.

For some values we need to provide them on our own machine which the hcloud cli is configured.

I wrote this script to provide value of `ETCD_INITIAL_CLUSTER` variable:

```shell
ARRAY=()
for NODE in $(hcloud server list --selector cluster=${CLUSTER_NAME} --selector etcd=true -o noheader -o columns=name); do
  ARRAY+=("${NODE}=https://$(hcloud server describe ${NODE} -o format='{{ (index .PrivateNet 0).IP}}'):2380")
done

ETCD_INITIAL_CLUSTER=$(printf ",%s" "${ARRAY[@]}")
ETCD_INITIAL_CLUSTER=${ETCD_INITIAL_CLUSTER:1}
echo $ETCD_INITIAL_CLUSTER
```

Run this on your own machine and use it when editing the etcd config file.

Also, you need each server internal ip. Make sure to use the correct value on each server.

Then, we need to uncomment and update these values in the `/etc/default/etcd` file.

* ETCD_NAME: use hostname value for each server.
* ETCD_INITIAL_CLUSTER: ${ETCD_INITIAL_CLUSTER}
* ETCD_INITIAL_ADVERTISE_PEER_URLS: https://${INTERNAL_IP}:2380
* ETCD_LISTEN_PEER_URLS: https://${INTERNAL_IP}:2380
* ETCD_LISTEN_CLIENT_URLS: https://${INTERNAL_IP}:2379,https://127.0.0.1:2379
* ETCD_ADVERTISE_CLIENT_URLS: https://${INTERNAL_IP}:2379
* ETCD_CLIENT_CERT_AUTH: true
* ETCD_PEER_CLIENT_CERT_AUTH: true
* ETCD_TRUSTED_CA_FILE: /etc/etcd/ca.crt
* ETCD_CERT_FILE: /etc/etcd/server.crt
* ETCD_KEY_FILE: /etc/etcd/server.key
* ETCD_PEER_TRUSTED_CA_FILE: /etc/etcd/ca.crt
* ETCD_PEER_CERT_FILE: /etc/etcd/peer.crt
* ETCD_PEER_KEY_FILE: /etc/etcd/peer.key

Use real values in the above variables.

Now restart etcd service on each server:

```shell
systemctl restart etcd.service
```

If it didn't restart successfully, check the etcd log with:

```shell
journalctl -xeu etcd.service
```

### Verify

To verify if it works enter this command on each server:

```shell
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/server.crt \
  --key=/etc/etcd/server.key
```

Output should be like:

```
11a85cd56d9530bf, started, node-1, https://10.1.1.1:2380, https://10.1.1.1:2379
6e6ca551d2049908, started, node-2, https://10.1.1.2:2380, https://10.1.1.2:2379
cba426eb7a695eef, started, node-3, https://10.1.1.3:2380, https://10.1.1.3:2379
```

You can store a value in a node:

```shell
ETCDCTL_API=3 etcdctl set foo bar \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/server.crt \
  --key=/etc/etcd/server.key
```

and check it on other nodes:

```shell
ETCDCTL_API=3 etcdctl get foo \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/server.crt \
  --key=/etc/etcd/server.key
```

It should show `bar` as value.
