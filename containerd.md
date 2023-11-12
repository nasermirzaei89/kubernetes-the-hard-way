# Install and Configure Containerd

You need to run containerd (or any other container runtime) on all worker nodes.

## Install Containerd

```shell
apt install -y containerd=1.7.2-0ubuntu1~22.04.1
```

Install an explicit version to prevent any issue with configuration change on new updates.

## Configure the `systemd` cgroup driver

To use the `systemd` cgroup driver in `/etc/containerd/config.toml` with `runc`,
we need to set it in containerd configuration.

```shell
mkdir -p /etc/containerd/
containerd config default | tee /etc/containerd/config.toml
```

Edit `/etc/containerd/config.toml` and set `SystemdCgroup = true`.

Then restart containerd.

```shell
systemctl restart containerd
```
