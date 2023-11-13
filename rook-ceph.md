# Install Rook

> You can't run it in a single small node cluster!

## Install Operator

```shell
helm upgrade --install rook-ceph --repo https://charts.rook.io/release rook-ceph \
  --namespace rook-ceph --create-namespace \
  --atomic \
  --version v1.12.7
```

## Add Volumes

Create some volumes and attach them to the worker nodes without mounting.

```shell
NODE_NAME=???
VOLUME_NAME=vol-$(head -c 16 /dev/urandom | base64 | tr -d '+/' | tr -dc 'a-z0-9' | head -c 10)
CLUSTER_NAME=cluster1
hcloud volume create --name ${VOLUME_NAME} --size 64 --server ${NODE_NAME} --label cluster=${CLUSTER_NAME}
```

## Install a Cluster

```shell
cat <<EOF | tee rook-ceph-cluster-overrides.yaml
toolbox:
  enabled: true
  resources:
    limits:
      cpu: "100m"
      memory: "128Mi"
cephClusterSpec:
  mon:
    count: 1 # Recommended to be 3
  mgr:
    count: 1 # By default is 2
  resources:
    mgr:
      requests:
        cpu: "100m"
        memory: "256Mi"
    mon:
      requests:
        cpu: "100m"
        memory: "256Mi"
    prepareosd:
      requests:
        cpu: "100m"
        memory: "50Mi"
    osd:
      requests:
        cpu: "100m"
        memory: "256Mi"
  storage:
    useAllNodes: false
    nodes:
      - name: "node-XXXXX"
        devices:
          - name: "sdb"
      - name: "node-YYYYY"
        devices:
          - name: "sdb"
      - name: "node-ZZZZZ"
        devices:
          - name: "sdb"
cephBlockPools:
  - name: ceph-blockpool
    spec:
      failureDomain: host
      replicated:
        size: 2
    storageClass:
      enabled: true
      name: ceph-block
      isDefault: true
      reclaimPolicy: Delete
      allowVolumeExpansion: true
      volumeBindingMode: "Immediate"
      mountOptions: []
      allowedTopologies: []
      parameters:
        imageFormat: "2"
        imageFeatures: layering
        csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
        csi.storage.k8s.io/provisioner-secret-namespace: "{{ .Release.Namespace }}"
        csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
        csi.storage.k8s.io/controller-expand-secret-namespace: "{{ .Release.Namespace }}"
        csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
        csi.storage.k8s.io/node-stage-secret-namespace: "{{ .Release.Namespace }}"
        csi.storage.k8s.io/fstype: ext4
cephFileSystems: []
cephObjectStores: []
EOF
```

```shell
helm upgrade --install rook-ceph-cluster --repo https://charts.rook.io/release rook-ceph-cluster \
  --namespace rook-ceph --create-namespace \
  --atomic \
  -f rook-ceph-cluster-overrides.yaml \
  --version v1.12.7
```
