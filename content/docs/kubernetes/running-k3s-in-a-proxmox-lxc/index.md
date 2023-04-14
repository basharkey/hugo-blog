---
title: Running K3s in a Proxmox LXC
---
# Running K3s in a Proxmox LXC

## Background
I only have a couple servers so in my case dedicating an entire system for Kubernetes was not an option. The choice was then left between using a VM or LXC. While using a VM would probably be the best choice in most scenarios, I had a unique requirement of needing to access a ZFS pool across both my Kubernetes single node cluster and other LXC containers on my Proxmox host.

One major advantage of LXCs is I can manage all my ZFS pools on my Proxmox host and then [mount specific host paths on my containers](https://pve.proxmox.com/wiki/Linux_Container#_bind_mount_points). This allows me to share a single ZFS pool with multiple LXCs all while not having to worry about NFS and the potential performance implications.

## Software
- Proxmox v7.3
- Debian LXC v11.6
- K3s v1.25.6

## Proxmox Configuration
After you have created a new privileged LXC (uncheck `Unprivileged container` in GUI) append the follwing to your container config:

`/etc/pve/lxc/<ct-id>.conf`
```
lxc.apparmor.profile: unconfined
lxc.cap.drop:
lxc.mount.auto: "proc:rw sys:rw"
lxc.cgroup2.devices.allow: c 10:200 rwm
```

## LXC Configuration
Kubernetes also requires access to `/dev/kmsg`. This can be done by creating a soft link of `/dev/console` to `/dev/kmsg`:
```
ln -s /dev/console /dev/kmsg
```

Some people recommend putting this command in your `/etc/rc.local` to ensure it is run on boot. I prefer to add it as an override config to my K3s systemd service. This means it is only run when the K3s service is started:

`/etc/systemd/system/k3s.service.d/override.conf`
```
[Service]
ExecStartPre=-/bin/ln -s /dev/console /dev/kmsg
```
{{< hint info >}}
Note the use of `ExecStartPre=-` the `-` tells systemd to still start the service even if the command fails. This ensures that you can restart `k3s.service` because when the `ln` command is executed a second time it will fail on restart as the link already exists.
{{< /hint >}}

## K3s
Now that the service override exists you can simply download and run the [K3s installation script](https://docs.k3s.io/quick-start):
```
curl -sfL https://get.k3s.io | sh -
```

When the installation script starts `k3s.service` systemd will ensure the `ln` command is run before Kubernetes is actually started.


# References
- https://gist.github.com/triangletodd/02f595cd4c0dc9aac5f7763ca2264185
- https://davegallant.ca/blog/2021/11/14/running-k3s-in-lxc-on-proxmox/
