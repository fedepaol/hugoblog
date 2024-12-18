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

## Running 
