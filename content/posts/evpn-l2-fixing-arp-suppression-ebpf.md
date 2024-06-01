---
title: "Fixing ARP suppression in Linux L2EVPN with eBPF"
date: 2024-06-01T12:14:10+02:00
categories: ["networking", "frr", "ebpf"]
description: Making ARP suppression work with eBPF in Linux with FRR
---

# Making ARP suppression work with pure L2 EVPN with Linux and FRR

In my [previous post]({{< relref "./evpn-l2-arp-suppression.md" >}}), I described the issues of making ARP suppression work with an L2 only EVPN topology,
with Linux and FRR.

Here I am going to describe how a small process running on each leaf can fix the problem.

## The problem

Here I'll quickly summarize the problem. Given a topology like the following:

```bash
                  ┌────────────────────────────────┐
                  │                                │
                  │                                │
                  │                                │
┌───────────┐ ARP │           ┌───────────────┐    │
│           │     ├──────┐    │               │    │
│  host2    │◄────┤ eth1 ├────┤     br-10     │    │
│           │     ├──────┘    │               │    │
└───────────┘     │           └──────────▲────┘    │
                  │                      │         │
                  │                      │         │
                  │                      │         │
                  │             leaf2    │         │
                  └──────────────────────┼─────────┘
                                         │
                                         │   VXLan
                                         │
                  ┌──────────────────────┼─────────┐
                  │                      │         │
                  │                      │         │
                  │                      │         │
┌───────────┐ARP  │           ┌──────────┴────┐    │
│           │     ├──────┐    │               │    │
│  host1    ├────►│ eth1 ├────┤     br-10     │    │
│           │     ├──────┘    │               │    │
└───────────┘     │           └───────────────┘    │
                  │                                │
                  │                                │
                  │                                │
                  │             leaf1              │
                  └────────────────────────────────┘
```


- being unicast, the ARP replies won't land on `leaf1`
- `leaf1`'s neighbor table is never filled
- the Zebra instance on the `leaf1` endpoint won't learn the MAC / IP association
- FRR won't send the MAC / IP association to `leaf2` as type 2 EVPN route
- ARP suppression on `leaf2` won't work, and every ARP request is going to be forwarded through the fabric

## Filling the neighbor table 

In my [previous post]({{< relref "./evpn-l2-arp-suppression.md" >}}), I explained how the root cause of ARP suppression not working
was the fact hat `leaf1` neihgbor table wasn't getting filled, and I showed how pinging `host1` from `leaf1`
would fill `leaf1`'s neighbor table, making all the machinery effectively work.

Additionally, this [issue comment on the FRR github repo](https://github.com/FRRouting/frr/issues/12574#issuecomment-1458953110) explains how
cumulus Linux solves this problem by listening at ARP requests and messing up with the local neighbor table.

## eBPF to the rescue!

Ebpf is not as cool as AI, but can certainly be more effective in this scenario. 
The plan is simple: write a program that leverages eBPF to listen to ARP requests / replies and fills the neighbor table from user space.

```bash

                        ┌────────────────────────────────┐
                        │                                │
                        │                 ┌────────────┐ │
                        │                 │ neigh table│ │
                        │                 │            │ │
                        │                 │            │ │
                        │                 │            │ │
                        │                 │            │ │
                        │    ┌─────────┐  └────────────┘ │
                        │  ┌►│  ebpf   │         ▲       │
                        │  │ └─────┬───┘         │       │
                        │  │       │       ┌─────┴─────┐ │
                        │  │       └──────►│ userspace │ │
                        │  │               └───────────┘ │
      ┌───────────┐ARP  │  │                             │
      │           │     │  │        ┌───────────────┐    │
      │  host1    ├────►├──┴───┐    │               │    │
      │           │     │ eth1 ├────┤     br-10     │    │
      └───────────┘     ├──────┘    │               │    │
                        │           └───────────────┘    │
                        │                                │
                        │                                │
                        │             leaf1              │
                        └────────────────────────────────┘

```

The poc can be found on [my github repo](https://github.com/fedepaol/fill-neighbors).

#### The eBPF part

The [eBPF part](https://github.com/fedepaol/fill-neighbors/blob/main/ebpf/tc_arplistener.c) is pretty simple (omitting all the checks to
make the verifier happy):

```C
  __u16 hlen = arph->ar_hln;
  __u16 plen = arph->ar_pln;
  __u32 opCode = bpf_ntohs(arph->ar_op);

  void *senderHwAddress = (void *)arph + sizeof(struct arphdr);
  void *senderProtoAddress = senderHwAddress + hlen;

  struct event *arp_event;

  arp_event = bpf_ringbuf_reserve(&events, sizeof(struct event), 0);

  bpf_core_read_str(arp_event->senderHWvalue, hlen + 1, senderHwAddress);
  bpf_core_read_str(arp_event->senderProtoValue, plen + 1, senderProtoAddress);
  arp_event->opCode = opCode;

  bpf_ringbuf_submit(arp_event, 0);
```

It gets the sender IP and MAC address from the ARP request, fills an event and sends it to user space via a ring buffer.

Simple and bulletproof.

#### On user space

The [user space side](https://github.com/fedepaol/fill-neighbors/commit/43e071c61510c55cc617c649933e963d4dac1c90#diff-2873f79a86c0d8b3335cd7731b0ecf7dd4301eb19a82ef7a1cba7589b5252261R78) is simple as well (with purged error handling):

```go
for {
		record, err := rd.Read()

		var event arpEvent
		binary.Read(bytes.NewBuffer(record.RawSample), binary.LittleEndian, &event)

		ip := bytesToIP(event.SenderProtoValue)
		mac := bytesToMac(event.SenderHWvalue)
		when := added[ip]
		now := time.Now()
		if now.Sub(when) > time.Minute {
			added[ip] = now
			neighborAdd(devID.Index, ip, mac)
		}
	}
 ```

 It:

 - receives the ARP related information from Kernel space
 - checks if it received one in the last minute
 - if not, adds the record to the neighbor table
 
 ## Does it work?

 Let's try with the same layout presented in [the l2 evpn post]({{< relref "./evpn-l2.md" >}}):

 When we start, the neigh table is does not contain the entries related to `HOST1` or `HOST2`:

```bash
docker exec -it clab-evpnl2-leaf1 ip neigh show
192.168.1.0 dev eth1 lladdr aa:c1:ab:94:bd:76 REACHABLE
fe80::a8bb:ccff:fe00:64 dev br10 lladdr aa:bb:cc:00:00:64 extern_learn NOARP proto zebra
```

We run the program on both the leaves:

```bash
leaf1:/# ./fill-neighbor --attach-to eth2 --from-interface br10
leaf2:/# ./fill-neighbor --attach-to eth2 --from-interface br10
```

And then we ping `HOST2` from `HOST1`:

```bash
docker exec clab-evpnl2-HOST1 bash -c "ping -c 1 192.168.10.3"
PING 192.168.10.3 (192.168.10.3) 56(84) bytes of data.
64 bytes from 192.168.10.3: icmp_seq=1 ttl=64 time=0.616 ms

--- 192.168.10.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.616/0.616/0.616/0.000 ms
```

Let's check the neighbor table of leaf1:

```bash
docker exec -it clab-evpnl2-leaf1 ip neigh show
192.168.10.2 dev br10 lladdr aa:c1:ab:3b:a1:fd STALE
192.168.1.0 dev eth1 lladdr aa:c1:ab:94:bd:76 STALE
192.168.10.3 dev br10 lladdr aa:c1:ab:69:b5:9e extern_learn NOARP proto zebra
fe80::a8bb:ccff:fe00:64 dev br10 lladdr aa:bb:cc:00:00:64 extern_learn NOARP proto zebra
```

`leaf1`'s neighbor table now contains:

- an entry related to `HOST1` (192.168.10.2), filled by the local `fill-neighbor` as an effect of the ARP request coming through eth2
- an entry related to `HOST2`, because `leaf2`'s local table was filled by its `fill-neighbor` as an effect of the ARP reply coming through its `eth2`, and then
FRR announced it as type 2 EVPN route to `leaf1`

Arping is resolved locally to leaf2, suppressing ARP broadcasts around the fabric.

 #### Things to improve

 This is a simple example, that clearly needs to be extended. It lacks IPV6 support, and more importantly the table must be kept in sync in two ways: 

 - by pinging the neighbor, to understand if the neighbor is still there and to refresh the local neigh table
 - by purging the entry of the map in case the neighbor is not there anymore

## Conclusion

This is yet another example of how eBPF programs can be very simple but enable powerful features of the kernel. Here we just observe
(and parse) the ARP requests, and use this information to keep the neighbor table using netlink commands.


