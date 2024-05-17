---
draft: true
title: "Some notes about ARP suppression in L2EVPN using FRR and Linux VXLans"
date: 2024-05-17T12:14:10+02:00
categories: ["networking", "frr"]
description: Details on ARP suppression in Linux with FRR and L2EVPN VXLans
---

# EVPN - ARP suppression

This post was meant to be a section of my previous [post about L2 tunnels with EVPN and VXLans]({{< relref "./evpn-l2.md" >}}),
but this particular topic kept me so busy that is probably worth a section of its own. Because of this, the topology this post
is referring to is still the same:


```bash
                      ┌─────────┐
                      │         │
       SPINE          │  64612  │
                      │         │
                      └────┬────┘
                           │
                           │
                   ┌───────┴────────┐
                   │                │
              ┌────┴────┐      ┌────┴────┐
              │         │      │         │
LEAVES        │  ┌───┐  │      │  ┌───┐  │
              │  │   │  │      │  │   ├──┼───────┐
              └──┴─┬─┴──┘      └──┴─┬─┴──┘       │
                   │                │            │    <-- L2
              ┌────┴────┐     ┌─────┴────┐  ┌────┴─────┐
              │         │     │          │  │          │
              │  Host1  │     │   Host2  │  │   Host3  │
              │         │     │          │  │          │
              └─────────┘     └──────────┘  └──────────┘

          192.168.10.2/24    192.168.10.3/24   192.168.10.4/24
```

## What ARP suppression is

Arp suppression is a mechanism where a given EVPN endpoint proxies the replies to ARP requests to avoid flooding the underlay with
multicast messages directed to all the VTEPs (virtual endpoints). The reply should be based on the information received via the
Type 2 EVPN routes.

## ARP suppression and FRR

The [FRR documentation](https://docs.frrouting.org/en/latest/evpn.html#linux-interface-configuration) say clearly that ARP suppression
is not FRR's responsibility:

```raw
    An SVI for an L2VNI is only needed for routing (IRB) or ARP/ND suppression.
    ARP/ND suppression is a kernel function, it is not managed  FRR.
    ARP/ND suppression is enabled per bridge_slave via neigh_suppress.
    ARP/ND suppression should only be enabled on vxlan interfaces.
```

The documentation of the `neigh_suppress` parameter is quite blurry:

__Controls whether neighbor discovery (arp and nd) proxy and suppression is enabled on the port. By default this flag is off.__

although, it doesn't tell how.

## Let's try it

Let's assign an IP to the bridges and see what's happening to the ARP requests.

Arping from HOST2 to HOST1 works:

```bash
docker exec clab-evpnl2-HOST2 arping -c 3 192.168.10.2
ARPING 192.168.10.2 from 192.168.10.3 eth1
Unicast reply from 192.168.10.2 [AA:C1:AB:99:2A:66]  0.613ms
Unicast reply from 192.168.10.2 [AA:C1:AB:99:2A:66]  0.569ms
Unicast reply from 192.168.10.2 [AA:C1:AB:99:2A:66]  0.583ms
Sent 3 probes (1 broadcast(s))
Received 3 response(s)
```

If we tcpdump leaf2 though:

```bash
16:57:27.874295 eth2  B   ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
16:57:27.874304 eth3  Out ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
16:57:27.874306 vni110 Out ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
16:57:27.874336 eth1  Out IP 100.65.0.2.52640 > 100.64.0.1.4789: VXLAN, flags [I] (0x08), vni 110
ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
16:57:27.874337 br10  B   ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
16:57:27.874377 eth1  In  IP 100.64.0.1.52640 > 100.65.0.2.4789: VXLAN, flags [I] (0x08), vni 110
ARP, Reply 192.168.10.2 is-at aa:c1:ab:99:2a:66, length 28
16:57:27.874381 vni110 P   ARP, Reply 192.168.10.2 is-at aa:c1:ab:99:2a:66, length 28
16:57:27.874382 eth2  Out ARP, Reply 192.168.10.2 is-at aa:c1:ab:99:2a:66, length 28
```

We can see that the ARP request is always forwarded through the tunnel instead of getting proxied locally.

## Type 2 EVPN Routes and IP Addresses

The association between the MAC and the IP should be propagated via the type 2 EVPN routes.

As we saw in the previous post though, the one received  leaf2 contain only
the MAC address learnt from the FIB of the leaf1's bridge (note that the iplen is 0 and the IP is not being set).

```bash
*> [2]:[0]:[48]:[aa:c1:ab:99:2a:66]
                      100.64.0.1                             0 64612 64512 i
                      RT:64512:110 ET:8
```

After a bit of digging and desperation, I learned that FRR's ZEBRA fills the information with the content of the local neighbor table.

At the same time, I checked the [tests of the kernel for the neigh_suppress option](https://github.com/torvalds/linux/blob/9a169c267e946b0f47f67e8ccc70134708ccf3d4/tools/testing/selftests/net/test_bridge_neigh_suppress.sh#L312)
(and tried to look at the source code as well).

In order fill the ARP table of `leaf1` with the IP of HOST1, we can arping HOST1 from `leaf1`:

```bash
sudo ip netns exec clab-evpnl2-leaf1 ping -I br10 -c 2 192.168.10.2
PING 192.168.10.2 (192.168.10.2) from 192.168.10.0 br10: 56(84) bytes of data.
64 bytes from 192.168.10.2: icmp_seq=1 ttl=64 time=0.088 ms
64 bytes from 192.168.10.2: icmp_seq=2 ttl=64 time=0.087 m
```

Now, looking at the EVPN routes again:

```bash

docker exec -it clab-evpnl2-leaf2 vtysh -c "show bgp l2vpn evpn" | grep -A2 192.168.10.2
 *> [2]:[0]:[48]:[aa:c1:ab:99:2a:66]:[32]:[192.168.10.2]
                    100.64.0.1                             0 64612 64512 i
                    RT:64512:110 ET:8
```

Now leaf2 is aware not only about the MAC of HOST1, but also of it's IP!

We also see that ZEBRA filled the neigh table of leaf2:

```bash
docker exec -it clab-evpnl2-leaf2 ip neigh show
192.168.1.2 dev eth1 lladdr aa:c1:ab:94:1f:14 REACHABLE
192.168.10.2 dev br10 lladdr aa:c1:ab:99:2a:66 extern_learn NOARP proto zebra
192.168.10.0 dev br10 lladdr aa:bb:cc:00:00:65 extern_learn NOARP proto zebra
```

Trying to arping HOST1 from HOST2 and checking what happens on leaf2, we can now see
that the ARP resolution happens without going through the underlay:

```bash
sudo ip netns exec clab-evpnl2-leaf2 tcpdump -nn -i any
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
17:25:20.521252 eth2  B   ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
17:25:20.521259 eth2  Out ARP, Reply 192.168.10.2 is-at aa:c1:ab:99:2a:66, length 28
```

As a counter-test, let's delete the neighbor entry from `leaf1` and check the neigh entry is removed both from
leaf1 and leaf2:

```bash
✗ docker exec clab-evpnl2-HOST2 arping -I eth1 -c 1 192.168.10.2

✗ docker exec -it clab-evpnl2-leaf2 ip neigh show
192.168.1.2 dev eth1 lladdr aa:c1:ab:94:1f:14 REACHABLE
192.168.10.0 dev br10 lladdr aa:bb:cc:00:00:65 extern_learn NOARP proto zebra
fe80::a8bb:ccff:fe00:65 dev br10 lladdr aa:bb:cc:00:00:65 extern_learn NOARP proto zebra

✗ docker exec -it clab-evpnl2-leaf1 ip neigh show
192.168.1.0 dev eth1 lladdr aa:c1:ab:8e:65:1b REACHABLE
fe80::a8bb:ccff:fe00:64 dev br10 lladdr aa:bb:cc:00:00:64 extern_learn NOARP proto zebra
```

If we tcpdump again while running arping from HOST2, we can see that now the ARP request
makes its way through the other host via the tunnel again:

```bash
sudo ip netns exec clab-evpnl2-leaf2 tcpdump -nn -i any
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
17:28:42.645234 eth2  B   ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
17:28:42.645241 eth3  Out ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
17:28:42.645242 vni110 Out ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
17:28:42.645258 eth1  Out IP 100.65.0.2.52640 > 100.64.0.1.4789: VXLAN, flags [I] (0x08), vni 110
ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
17:28:42.645262 br10  B   ARP, Request who-has 192.168.10.2 (ff:ff:ff:ff:ff:ff) tell 192.168.10.3, length 28
17:28:42.645329 eth1  In  IP 100.64.0.1.52640 > 100.65.0.2.4789: VXLAN, flags [I] (0x08), vni 110
ARP, Reply 192.168.10.2 is-at aa:c1:ab:99:2a:66, length 28
17:28:42.645334 vni110 P   ARP, Reply 192.168.10.2 is-at aa:c1:ab:99:2a:66, length 28
17:28:42.645335 eth2  Out ARP, Reply 192.168.10.2 is-at aa:c1:ab:99:2a:66, length 28
```

## Conclusion

In order to have IPs in Type-2 routes (and consequently, ARP suppression to work),
the Neighbor table of the leaf connected to a host must be filled with the MAC and the IP of that host.

In case of a pure L2 topology this won't happen, as the ARP reply message is unicast and directed to
the HOST's interface.

What I showed here is just an example to show the mechanics of ARP suppression (and Type 2 routes with IP)
with FRR and Linux. My understanding is that there are scenarios where this is worked around by
forcing those neighbor entries (such as Cumulus Linux with `neighmgrd`).
