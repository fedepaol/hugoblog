---
title: "L2EVPN using FRR and Linux VXLans"
date: 2024-05-17T12:14:10+02:00
categories: ["networking", "frr"]
description: How to configure FRR and Linux VXLans to implement EVPN L2 tunnels
---

# EVPN - L2 tunnels

This is a follow up of the previous [post about L3 tunnels with EVPN and VXLans]({{< relref "./evpn-l3.md" >}}).

I started with L3 because, as much as counter intuitive it was (since VXLan is about tunneling L2 links), it felt more similar to what
I am used to with BGP.

In [my evpn lab github repository](https://github.com/fedepaol/evpnlab/tree/main/02_clab_l2) I added a new entry where I
set up a L2 only topology using FRR and Containerlab.

## The topology

The topology is quite similar to what I described in [the first post of this serie]({{< relref "./evpn-l3.md" >}}), with the only difference
being that the links between the HOSTs and the leaves are L2 only, and the veth leg corresponding to each connection is enslaved directly to the bridge inside
each leaf.


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

This setup relies on Containerlab, so I won't describe again the details as I did in the first part.

The ContainerLab configuration can be found [here](https://github.com/fedepaol/evpnlab/blob/main/02_clab_l2/l2.clab.yml).

## Setting up the hosts

Setting up the hosts is easy. Since we want all of them to belong to the same subnet, we just assign the IP to each host's interface.

```bash
ip addr add 192.168.10.2/24 dev eth1

ip r del default
sleep INF
```

## Setting up the leaves

The frr configuration remains the same. The leaves are connect to the spine and announce both the VTEP ips via bgp and
the EVPN address family.

What is different is the host setup:

The VXLan's VNI (110) is different from the one associated to the IP-VRF declared in the frr configuration (100). On top of that, since this
is a pure L2 setup, we don't assign any IP to the bridge.

Finally, we enslave the interface connected to the host directly to the bridge.

```bash
ip link add br10 type bridge
ip link set br10 master red
ip link set br10 addr aa:bb:cc:00:00:65
ip link add vni110 type vxlan local 100.64.0.1 dstport 4789 id 110 nolearning
ip link set vni110 master br10 addrgenmode none
ip link set vni110 type bridge_slave neigh_suppress on learning off
ip link set vni110 up
ip link set br10 up
ip link set eth2 master br10
```

## Bringing it up

As per the previous case, a convenience [setup script](https://github.com/fedepaol/evpnlab/blob/main/02_clab_l2/setup.sh) is provided.

## Making sure everything works

Instead of running the ping test (which works as I'll show later) I want to show the mechanics of L2EVPN as they are probably more interesting than
the L3 ones.

Being a L2 device, each bridge learns about the mac address of the connected interfaces. Given the MAC address of the `eth1` interface of `HOST1`:

```bash
docker exec -it clab-evpnl2-HOST1 ip l | grep -A1 eth1
351: eth1@if350: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 qdisc noqueue state UP mode DEFAULT group default
    link/ether aa:c1:ab:99:2a:66 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

We can see it as an fdb entry of the `br10` bridge we enslaved the interface to on leaf1:

```bash
docker exec -it clab-evpnl2-leaf1 bridge fdb show br10 | grep aa:c1:ab:99:2a:66
aa:c1:ab:99:2a:66 dev eth2 master br10
```

This doesn't tell anything new. What's more interesting is finding the same MAC address on the bridge of leaf2:

```bash
docker exec -it clab-evpnl2-leaf2 bridge fdb show br10 | grep aa:c1:ab:99:2a:66
aa:c1:ab:99:2a:66 dev vni110 vlan 1 extern_learn master br10
aa:c1:ab:99:2a:66 dev vni110 extern_learn master br10
aa:c1:ab:99:2a:66 dev vni110 dst 100.64.0.1 self extern_learn
```

Leaf2 learned about the HOST1's MAC address and knows it's reacheable via the `100.64.0.1` VTEP through the `vni110`
VXLan.

Looking at what we have in FRR, we can see that the leaf2 instance learned about the MAC address associated
to the remote VTEP.

```bash
docker exec -it clab-evpnl2-leaf2 vtysh -c "show evpn mac vni all"

VNI 110 #MACs (local and remote) 4

Flags: N=sync-neighs, I=local-inactive, P=peer-active, X=peer-proxy
MAC               Type   Flags Intf/Remote ES/VTEP            VLAN  Seq #'s
aa:c1:ab:21:6e:91 local        eth2                                 0/0
aa:bb:cc:00:00:64 local        br10                           1     0/0
aa:c1:ab:99:2a:66 remote       100.64.0.1                           0/0
aa:bb:cc:00:00:65 remote       100.64.0.1                           0/0
```

Additionally, we can see the type 2 route related to the HOST1's MAC address:

```bash
docker exec -it clab-evpnl2-leaf2 vtysh -c "show bgp l2vpn evpn"
BGP table version is 5, local router ID is 100.65.0.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-1 prefix: [1]:[EthTag]:[ESI]:[IPlen]:[VTEP-IP]:[Frag-id]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 100.64.0.1:2

[.....]

*> [2]:[0]:[48]:[aa:c1:ab:99:2a:66]
                      100.64.0.1                             0 64612 64512 i
                      RT:64512:110 ET:8

[.....]
```

So, the Zebra instance running on leaf2 learned about the bridge's FDB table and sent it over
MP-BGP to leaf1. The Zebra instance on the other side changed the local bridge to be aware of that
MAC (or something along the line) and everything works.

Let's now do the ping test:

```bash
docker exec clab-evpnl2-HOST2 ping -I eth1 -c 1 192.168.10.2
PING 192.168.10.2 (192.168.10.2) from 192.168.10.3 eth1: 56(84) bytes of data.
64 bytes from 192.168.10.2: icmp_seq=1 ttl=64 time=0.142 ms

--- 192.168.10.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.142/0.142/0.142/0.000 ms
```

It works! Let's ensure that it goes through the tunnel by tcpdumping in leaf2:

```bash
sudo ip netns exec clab-evpnl2-leaf2 tcpdump -nn -i any
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
17:33:21.052689 eth2  P   IP 192.168.10.3 > 192.168.10.2: ICMP echo request, id 22, seq 1, length 64
17:33:21.052706 vni110 Out IP 192.168.10.3 > 192.168.10.2: ICMP echo request, id 22, seq 1, length 64
17:33:21.052714 eth1  Out IP 100.65.0.2.33004 > 100.64.0.1.4789: VXLAN, flags [I] (0x08), vni 110
IP 192.168.10.3 > 192.168.10.2: ICMP echo request, id 22, seq 1, length 64
17:33:21.052764 eth1  In  IP 100.64.0.1.33004 > 100.65.0.2.4789: VXLAN, flags [I] (0x08), vni 110
IP 192.168.10.2 > 192.168.10.3: ICMP echo reply, id 22, seq 1, length 64
17:33:21.052764 vni110 P   IP 192.168.10.2 > 192.168.10.3: ICMP echo reply, id 22, seq 1, length 64
17:33:21.052775 eth2  Out IP 192.168.10.2 > 192.168.10.3: ICMP echo reply, id 22, seq 1, length 64
^C
6 packets captured
```

Here we can see:

- the packet coming in from eth2
- the packet leaving through the vxlan interface
- the encapsulated packet leaving towards the spine, via eth1


## Conclusion

This was another example on how to setup L2 EVPN with FRR, using containerlab as a testbed.
In a follow up post I will talk about ARP suppression and then will try to have something more
elaborate with two L2 domains belonging to the same VRF.
