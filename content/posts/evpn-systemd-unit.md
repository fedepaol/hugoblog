---
title: "Evpn Systemd Unit"
date: 2024-12-17T21:33:24+01:00
draft: true
---

# Quick Recap

This is a follow up to my lenghty EVPN series. In my [last post]({{< relref "./extending-kind-evpn.md" >}}) I demonstrated how I managed to implement a PE Router running locally to a kubernetes cluster,
hosted inside a regular pod and accessible from the host via veth pairs:

![](/images/evpnkindpods/packetpath-pods.png)

## Overcoming Limitations

I also described how running inside a pod is limiting my prototype to be used as the main access to the fabric, because of the chicken egg-y issue of needing an underlay for being able to start the pod in first instance.

In this post, I am going to take a step further by **running the two containers as systemd units**. The architecture is more or less the same, with the biggest difference of not having pods but just containers managed by podman.

![](/images/podman-systemd-unit/routerkind-podmanunits.png)

## Running podman pods as systemd units

The idea is still the same: one container with its own network namespace and running FRR, connected to the leaf router via an interface moved inside the namespace, and connected to the host with veth legs.
Another container with more powers, running as `network=host` in order to be able to create and move around the required interfaces.

The only difference here is that instead of running them as pod containers, we'll run them as podman pods containers, installed as systemd units.

The right enchantment can be found in various posts, and boils down to:

Create the pod:

```bash
podman pod create --share=+pid --name=frrpod -p 7080:7080
```

Create the containers belonging to the pod:

```bash
podman create --pod=frrpod --pidfile=/root/frr/frrpid --name=frr --cap-add=CAP_NET_BIND_SERVICE,CAP_NET_ADMIN,CAP_NET_RAW,CAP_SYS_ADMIN -v=/root/frr:/etc/frr -v=varfrr:/var/run/frr:Z -t quay.io/frrouting/frr:10.0
.1 /etc/frr/start.sh
podman create --pod=frrpod --name=reloader -v=/root/frr:/etc/frr -v=varfrr:/var/run/frr:Z --entrypoint=/etc/frr/reloader.sh -t quay.io/frrouting/frr:10.0.1
```

Generate the corresponding systemd units:

```bash
podman generate systemd --new --files --name frrpod
```

Starting the systemd unit:

```bash
systemctl start pod-controllerpod
```

By doing this, we give up the configurability via CRD offered by Kubernetes, but we gain independence from a kubernetes node lifecycle. In particular, this systemd units can be enabled and those containers can be started as soon as a node starts.

## Rethinking the architecture

This was a chance to rethink the architecture of the whole machinery, by moving the responsability of configuring the router container's network interfaces to the controller container.

## First hurdle: finding the right network namespace
