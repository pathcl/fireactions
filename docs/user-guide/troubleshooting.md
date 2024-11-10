# Troubleshooting

## How to access the virtual machine of GitHub runner?

To access the virtual machine, the ID of the virtual machine must be known. The ID can be found in GitHub Actions logs. Once the ID is known, the virtual machine can be accessed by finding the network namespace:

```bash
ip netns ls | grep e520c86a3ce850b055ed8f8e
fireactions-bnk-c8-m16-e520c86a3ce850b055ed8f8e (id: 3)
```

Then, find the IP address of the virtual machine:

```bash
ip netns exec fireactions-bnk-c8-m16-e520c86a3ce850b055ed8f8e ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0@if20202: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ea:32:bb:c6:49:0a brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.129.59/23 brd 192.168.129.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::e832:bbff:fec6:490a/64 scope link
       valid_lft forever preferred_lft forever
3: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether b6:cd:19:ae:29:b7 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::b4cd:19ff:feae:29b7/64 scope link
       valid_lft forever preferred_lft forever
```

In this case, the assigned IP address is `192.168.129.59`. The virtual machine can be accessed using SSH:

```bash
ssh -l root 192.168.129.59
```

The default password is `fireactions`. This can be changed by creating a custom image. Once logged in, the virtual machine can be managed as a regular Linux machine. Inside you will find `fireactions` service logs:

```bash
root@localhost:~# systemctl status fireactions
● fireactions.service - Fireactions
     Loaded: loaded (/etc/systemd/system/fireactions.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2024-09-27 14:54:48 UTC; 2 weeks 5 days ago
<...>
```

## containerd: creating snapshot: snapshotter not loaded: devmapper: invalid argument

If the following error is found in the logs:

```text
Nov 10 19:01:47 eu-lt-sng3-node3 fireactions[14729]: 2024-11-10 19:01:47 ERR pool.go:125 > Failed to scale pool error="containerd: creating snapshot: snapshotter not loaded: devmapper: invalid argument" pool=fireactions-2vcpu-2gb
```

This error is caused by the `devmapper` snapshotter not being loaded. To fix this, the `devmapper` snapshotter must be loaded. This can be done by adding the following line to the `containerd` configuration file:

```bash
tee /etc/containerd/config.toml <<EOF
[plugins]
  [plugins."io.containerd.snapshotter.v1.devmapper"]
    pool_name       = "containerd-thinpool"
    root_path       = "/var/lib/containerd/devmapper"
    base_image_size = "30GB"
    discard_blocks  = true
EOF

systemctl restart containerd
```

## containerd: creating snapshot: prepare: failed to create snapshot

If the following error is found in the logs:

```text
Nov 10 19:02:04 eu-lt-sng3-node3 fireactions[14729]: 2024-11-10 19:02:04 ERR pool.go:125 > Failed to scale pool error="containerd: creating snapshot: prepare: failed to create snapshot \"containerd-thinpool-snap-35\" (dev: 35) from \"containerd-thinpool-snap-23\" (dev: 23): no data available: unknown" pool=fireactions-2vcpu-2gb
```

The error message is not clear. This could happen when tampering with the installation. We've only been able to fix this by completely wiping Containerd namespaces and restarting the service. This is not a good solution, but it works.

To do this, run the following commands:

```bash
systemctl stop containerd

ctr namespaces ls
NAME                  LABELS
fireactions-2vcpu-4gb

ctr --namespace=fireactions-2vcpu-4gb snapshots --snapshotter=devmapper ls
KEY                                                                     PARENT                                                                  KIND
fireactions-2vcpu-4gb-943101bc9e5079d2886dc0ec                          sha256:7bcb2dfc39edf0d70dcee6285aa9cb9d5cab4d84933c454d69bfd811a6a477b5 Active

ctr --namespace=fireactions-2vcpu-4gb snapshots --snapshotter=devmapper rm fireactions-2vcpu-4gb-943101bc9e5079d2886dc0ec

ctr --namespace=fireactions-2vcpu-4gb i ls
REF                                                     TYPE                                                      DIGEST                                                                  SIZE      PLATFORMS               LABELS
ghcr.io/hostinger/fireactions-images/ubuntu22.04:v0.7.0 application/vnd.docker.distribution.manifest.list.v2+json sha256:c7dd9a4dd58300040a24a00c52a2035e1d428aeab95fbd81690a6bf37aeea87f 617.7 MiB linux/amd64,linux/arm64 -

ctr --namespace=fireactions-2vcpu-4gb i rm ghcr.io/hostinger/fireactions-images/ubuntu22.04:v0.7.0

ctr namespaces rm fireactions-2vcpu-4gb

systemctl start containerd
```
