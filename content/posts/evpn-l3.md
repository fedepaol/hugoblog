---
title: "L3EVPN using FRR and Linux VXLans"
date: 2024-05-09T12:01:51+01:00
categories: ["networking", "frr"]
description: How to configure FRR and Linux VXLans to implement EVPN L3 tunnels
---

# Tinkering with EVPN

EVPN and VXLan tunnels (and how FRR gets along with those) are something I wanted to explore, so
I thought it'd make sense to document my journey as a serie of blogposts (which can also serve as a help to my memory).

The result of this first rambling is available on my [github repository](https://github.com/fedepaol/evpnlab/tree/main/01_clab_l3).

## The topology

The first attempt is to have a spine - leaves topology with two hosts connected to two different leaves
via a L3 connection.


```bash
                     ┌─────────┐
                     │         │
                     │  64612  │
                     │         │
                     └────┬────┘
                          │
                          │
                  ┌───────┴────────┐
                  │                │
             ┌────┴────┐      ┌────┴────┐
             │         │      │         │
             │  64512  │      │  64512  │
             │         │      │         │
             └────┬────┘      └────┬────┘
192.168.10.0/24   │                │ L3     192.168.11.0/24
             ┌────┴────┐      ┌────┴────┐
             │         │      │         │
             │  Host1  │      │  Host2  │
             │         │      │         │
             └─────────┘      └─────────┘
```


We want to connect the two hosts via a VXLan tunnel using L3Evpn routes:

```bash


                  ┌─────────┐              ┌─────────┐
                  │         │              │         │
                  │  64512  │     VXLan    │  64512  │
                  │       ┌─┼──────────────┼─┐       │
                  └────┬──┴─┘              └─┴──┬────┘
     192.168.10.0/24   │                        │ L3     192.168.11.0/24
                  ┌────┴────┐              ┌────┴────┐
                  │         │              │         │
                  │  Host1  │              │  Host2  │
                  │         │              │         │
                  └─────────┘              └─────────┘
```

## ContainerLab

In order to test the configuration on my laptop, I am using [containerlab](https://containerlab.dev/), which provides a
nice way to setup any topology using just containers. The main difference from using straight docker-compose, is that the various containers implementing
the routing devices are connected together via veth pairs, making the whole lab more adherent to the reality.

Additionally, we can choose from a variety of implementations, but for the time being I am interested in [FRR](https://docs.frrouting.org).

### The container lab configuration

The configuration is pretty straightforward. We define nodes, where we can map files inside the containers:

```yaml
topology:
  nodes:
    leaf1:
      kind: linux
      image: quay.io/frrouting/frr:9.0.0
      binds:
        - leaf1/daemons:/etc/frr/daemons
        - leaf1/frr.conf:/etc/frr/frr.conf
        - leaf1/vtysh.conf:/etc/frr/vtysh.conf
        - leaf1/setup.sh:/setup.sh
```

and a set of links, to connect the nodes:

```yaml
  links:
    - endpoints: ["leaf1:eth1", "spine:eth1"]
    - endpoints: ["leaf2:eth1", "spine:eth2"]
    - endpoints: ["HOST1:eth1", "leaf1:eth2"]
    - endpoints: ["HOST2:eth1", "leaf2:eth2"]
```

**Note how eth0 is not mentioned**, as it's the default interface connecting the containers to the docker bridge, which serves no purpouses for our
topology.

### Setting up the hosts

Although Containerlab creates the veth pairs for us, it does not assign the IP to each leg:

```bash
ip addr add 192.168.10.1/24 dev eth1

# set the default gw via eth1
ip r del default
ip r add default via 192.168.10.2
sleep INF
```

On top of that, we set the default route to go through the leaf and not through the docker bridge.

Additionally, given no entry point is defined for our "plain linux" container, we can embed the
entry point directly in the clab declaration.

### Setting up the leaves

FRR [has a strong opinion](https://docs.frrouting.org/en/latest/evpn.html) in how the host layout should look like in order to support EVPN, delegating the responsibility of the tunneling part to the linux kernel. I won't repeat what the FRR documentation describes in detail, but will list the required steps:

- assign the VTEP IP address to the loopback interface
- Assign IPs to both the veth connecting the leaf to the spine and the one connecting the "HOST" to the leaf
- Create a linux VRF corresponding to the L3 VRF and enslave the veth leg connected to the "HOST"
- Create all the machinery to make the EVPN / VXLan tunnel work, including a linux bridge and a VXLan interface

### The leaves FRR configuration

The leaves FRR configuration resembles pretty much what the [FRR documentation](https://docs.frrouting.org/en/latest/evpn.html) describes. The difference
is where we allow the leaves to accept routes coming from the local ASN (from the spine):

```bash
 address-family l2vpn evpn
  neighbor 192.168.1.0 activate
  neighbor 192.168.1.0 allowas-in origin
  advertise-all-vni
  advertise-svi-ip
 exit-address-family
```

Also, note how we enable allowas-in for both the l2vpn and ip address families, as need to receive both the route to the VTEP hosted on the other leaf and the Type 5 EVPN routes advertised by the other leaves.

Additionally, we redistribute the connected routes so each leaf can distribute the route directed to the hosts:

```bash
router bgp 64512 vrf red
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
```


### The spine FRR configuration

The spine configuration is pretty simple. It's ASN is different from the one of the leaves, so no route reflection mechanism is needed.
The spine router must redirect the routes towards the VTEPs (for the underlay network) and the EVPN Type 5 routes.

```bash
 neighbor 192.168.1.1 remote-as 64512
 neighbor 192.168.1.3 remote-as 64512

 address-family ipv4 unicast
  neighbor 192.168.1.1 activate
  neighbor 192.168.1.3 activate
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor 192.168.1.1 activate
  neighbor 192.168.1.3 activate
  advertise-all-vni
  advertise-svi-ip
 exit-address-family
```

## Bringing it up

A convenience [setup.sh](https://github.com/fedepaol/evpnlab/blob/main/01_clab_l3/setup.sh) script is provided:
it brings up the lab and then runs the extra commands required to assign ips and create the vxlan setup in the leaves.

## Troubleshooting VXLan and EVPN

I made an incredible number of fat-finger related mistakes, so here I will try to reproduce some common troubleshooting
steps to make sure both the underlay network and the overlay network work.

Things to check on both the leaves and the spine include:

Checking the BGP routes for the underlay network:

```bash
leaf1:/# vtysh -c "show bgp ipv4"
BGP table version is 2, local router ID is 100.64.0.1, vrf id 0
Default local pref 100, local AS 64512
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

    Network          Next Hop            Metric LocPrf Weight Path
 *  100.64.0.1/32    192.168.1.0                            0 64612 64512 i
 *>                  0.0.0.0                  0         32768 i
 *> 100.65.0.2/32    192.168.1.0                            0 64612 64512 i

Displayed  2 routes and 3 total paths
```

Checking the VNI setup:

```bash
leaf1:/# vtysh -c "show vrf vni"
VRF                                   VNI        VxLAN IF             L3-SVI               State Rmac
red                                   100        vni100               br100                Up    aa:bb:cc:00:00:65
l
```

Checking the status of the session:

```bash
leaf1:/# vtysh -c "show bgp l2vpn evpn summary"
BGP router identifier 100.64.0.1, local AS number 64512 vrf-id 0
BGP table version 0
RIB entries 3, using 576 bytes of memory
Peers 1, using 13 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
192.168.1.0     4      64612       153       153        1    0    0 02:25:21            1        2 N/A

Total number of neighbors 1
```

Checking the type5 routes:

```bash
leaf1:/# vtysh -c "show bgp l2vpn evpn"
BGP table version is 1, local router ID is 100.64.0.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-1 prefix: [1]:[EthTag]:[ESI]:[IPlen]:[VTEP-IP]:[Frag-id]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 192.168.10.2:2
 *> [5]:[0]:[24]:[192.168.10.0]
                    100.64.0.1               0         32768 ?
                    ET:8 RT:64512:100 Rmac:aa:bb:cc:00:00:65
Route Distinguisher: 192.168.11.2:2
 *> [5]:[0]:[24]:[192.168.11.0]
                    100.65.0.2                             0 64612 64512 ?
                    RT:64512:100 ET:8 Rmac:aa:bb:cc:00:00:64

Displayed 2 out of 2 total prefixes
```

If it's still not clear why routes are not going through, I strongly suggest to have a look at the FRR logs. Despite intimidating, they offered the solution straight away the majority of the cases.

This included having selected the same mac address for the Linux bridge on both leaves, and not allowing routes for the local ASN for the l2vpn address family.

## Checking if it actually works

If we try to ping HOST2 from HOST1:

```bash
bash-5.1# ping 192.168.11.1
PING 192.168.11.1 (192.168.11.1) 56(84) bytes of data.
64 bytes from 192.168.11.1: icmp_seq=1 ttl=62 time=0.321 ms
64 bytes from 192.168.11.1: icmp_seq=2 ttl=62 time=0.198 ms
```

It works!

Now, if we check what's happening on the link between the host and the leaf:


```bash
                     ┌─────────┐
                     │         │
                     │  64612  │
                     │         │
                     └────┬────┘
                          │
                          │
                  ┌───────┴────────┐
                  │                │
             ┌────┴────┐      ┌────┴────┐
             │         │      │         │
             │  64512  │      │  64512  │  <---- LEAF
             │         │      │         │
             └────┬────┘      └────┬────┘
192.168.10.0/24   │                │ L3     <---- HERE
             ┌────┴────┐      ┌────┴────┐
             │         │      │         │
             │  Host1  │      │  Host2  │
             │         │      │         │
             └─────────┘      └─────────┘
```

```bash
# sudo ip netns exec clab-evpnl3-leaf1 tcpdump -nn -i eth2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
12:04:37.836699 IP 192.168.10.1 > 192.168.11.1: ICMP echo request, id 6, seq 5, length 64
12:04:37.836868 IP 192.168.11.1 > 192.168.10.1: ICMP echo reply, id 6, seq 5, length 64
```

Probably more interesting is what's happening on the leg connecting the leaf to the spine:

```bash
                     ┌─────────┐
                     │         │
                     │  64612  │
                     │         │
                     └────┬────┘
                          │
                          │
                  ┌───────┴────────┐
                  │                │   <---- HERE
             ┌────┴────┐      ┌────┴────┐
             │         │      │         │
             │  64512  │      │  64512  │ <---- LEAF
             │         │      │         │
             └────┬────┘      └────┬────┘
192.168.10.0/24   │                │ L3
             ┌────┴────┐      ┌────┴────┐
             │         │      │         │
             │  Host1  │      │  Host2  │
             │         │      │         │
             └─────────┘      └─────────┘
```

We see that we are sending a VXLan packet directed to the VTEP on the other leaf embedding the ICMP request:

```bash
sudo ip netns exec clab-evpnl3-leaf1 tcpdump -nn -i eth1
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
12:01:51.759318 IP 100.64.0.1.48561 > 100.65.0.2.4789: VXLAN, flags [I] (0x08), vni 100
IP 192.168.10.1 > 192.168.11.1: ICMP echo request, id 5, seq 1, length 64
12:01:51.759433 IP 100.65.0.2.48561 > 100.64.0.1.4789: VXLAN, flags [I] (0x08), vni 100
IP 192.168.11.1 > 192.168.10.1: ICMP echo reply, id 5, seq 1, length 64
```

# Conclusion

ContainerLab made it super easy to validate topologies and configurations, and thanks to that I managed (not without obstacles) to setup a proper
spine leaves topology with EVPN. Next step is to add L2Evpn tunneling, and possibly replacing the eBGP spine with a route reflector.

Also, as a beginner, I am pretty sure there are better ways to implement the same result. If you think it can be improved, please reach out or file a pull request!
