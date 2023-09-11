---
title: "Debugging XDP"
date: 2023-09-06T23:26:17+02:00
draft: true
---

# Overcoming issues with XDP

This is a side post from my "learning eBPF from popular projects" initiative. Here I am going to describe the issues I had while trying to replicate
locally an XDP based loadbalancer (see my [previous post]({{< ref "/posts/ebpf-journey-loadbalancing-with-katran" >}})).

## What happened

Right after making the verifier gods happy, I started trying to replicate an environment I could test it in.

The setup was docker-compose based, and was _composed_ by multiple containers and docker networks tied together:

```raw
              10.111.220.0/24                 10.111.222.0/24

┌──────────────────┐       ┌───────────────────┐        ┌───────────────────┐
│                  │       │                   │        │                   │
│                  │ ◄─────┼───────────────────┼────────┤                   │
│                  │       │                   │        │                   │
│    Client        ├──────►│  Gateway          │        │    Real           │
│                  │       │             ┌─────┼───────►│                   │
│                  │       │             │     │        │                   │
│                  │       │             │     │        │                   │
└──────────────────┘       └──────┬──────┴─────┘        └───────────────────┘
                                  │      ▲
                                  │      │
                                  │      │  10.111.221.0/24
                                  ▼      │
                            ┌────────────┴─────┐
                            │                  │
                            │                  │
                            │                  │
                            │     Katran       │
                            │                  │
                            │                  │
                            └──────────────────┘
```

So things were smooth, I was able to ping, the routes were in place, and I was ready to try the full
workflow, using netcat on the client to reach the endpoint via the virtual IP.

I spun up nc on both sides, tried to send a string, and... nothing. Which was kind of expected.
After all, the program messes up with the raw packets, dealing with a lot of offset and low level fields.

The chance of getting it right in first instance were pretty low, so I took my old `tcpdump`
friend out of my pocket and started debugging.

### The gateway was routing the syn packet to katran

```bash
21:30:45.998533 eth1  Out IP 10.111.220.11.35562 > 192.168.10.1.50002: Flags [S], seq 33054345, win 64240, options [mss 1460,sackOK,TS val 2254957155 ecr 0,nop,wscale 7], length 0
21:30:47.015935 eth0  In  IP 10.111.220.11.35562 > 192.168.10.1.50002: Flags [S], seq 33054345, win 64240, options [mss 1460,sackOK,TS val 2254958173 ecr 0,nop,wscale 7], length 0
21:30:47.015953 eth1  Out IP 10.111.220.11.35562 > 192.168.10.1.50002: Flags [S], seq 33054345, win 64240, options [mss 1460,sackOK,TS val 2254958173 ecr 0,nop,wscale 7], length 0
21:30:49.063738 eth0  In  IP 10.111.220.11.35562 > 192.168.10.1.50002: Flags [S], seq 33054345, win 64240, options [mss 1460,sackOK,TS val 2254960221 ecr 0,nop,wscale 7], length 0
21:30:49.063756 eth1  Out IP 10.111.220.11.35562 > 192.168.10.1.50002: Flags [S], seq 33054345, win 64240, options [mss 1460,sackOK,TS val 2254960221 ecr 0,nop,wscale 7], length 0
```

I could see the packet coming from the client and going out to eth1 (the interface with the loadbalancer).

And this was only the beginning.

## Running tcpdump on the loadbalancer container does not show anything

Nothing, **dead silence**. Who stole my packets?

I was not sure that the paramters I was passing to the program were correct. In fact, they weren't.

## Logging everywhere!

`bpf_printk` is the more convenient way to print something you can inspect from `/sys/kernel/debug/tracing/trace_pipe`.
That allowed me to undersand in first instance that the VIP I was reading from eBPF was empty.

## Checking the content of the map

`bpftool` is the swiss knife here. 

Accessing the map is as easy as running 

```bash
-> bpftool map list 
...
86: array  name xdp_params_arra  flags 0x0
        key 4B  value 20B  max_entries 1  memlock 344B
        btf_id 236
        pids xdplb(10671)
...
```

And then

```bash
bpftool map dump id 86 
[{
        "key": 0,
        "value": {
            "dst_mac": [2,66,10,111,221,12
            ],
            "daddr": 0,
            "saddr": 175103243,
            "vip": 0
        }
    }
]
```

it was clear that some parameters were not going through properly, I hammered the Go code a bit and made them work. And, as always,
I was confident this would have made things work. But it did not.

### XDPDump to the rescue

My Google fu brought me to [xdpdump](https://github.com/xdp-project/xdp-tools/tree/master/xdp-dump). XDP dump is a bit like TCPDump, but it
is able to inspect the packets going through (ingressing and egressing) an XDP program. There is a nice blogpost about it [on the Red Hat blog](https://www.redhat.com/en/blog/capturing-network-traffic-express-data-path-xdp-environment). 
So, by running

```bash
xdpdump --rx-capture entry,exit --i eth0

1694209748.510390618: xdp_prog_func()@entry: packet size 74 bytes on if_index 15, rx queue 0, id 1
1694209748.510401404: xdp_prog_func()@exit[TX]: packet size 94 bytes on if_index 15, rx queue 0, id 1
1694209749.530575406: xdp_prog_func()@entry: packet size 74 bytes on if_index 15, rx queue 0, id 2
1694209749.530604456: xdp_prog_func()@exit[TX]: packet size 94 bytes on if_index 15, rx queue 0, id 2
```

I could see that something was actually happening. In particular, packets were coming to the program (`xdp_prog_func`), and getting the `TX`
verdict. At least my program was getting invoked. This was also confirmed by many of the logs I put in place. Still, no packet :-(

XDPDump allows us to extract the pcap (and eventually consume it with wireshark) which is what I did:

```bash
xdpdump --rx-capture entry,exit --i eth0 -w /data/capture.pcap
listening on eth0, ingress XDP program ID 180 func xdp_prog_func, capture mode entry/exit, capture size 262144 bytes
^C
6 packets captured
0 packets dropped by perf rin
```

Then I opened wireshark, saw a malformed packet (I messed up with the lenght of the IP Header), I fixed it and full of hope I did
the test again. And it did not work.

This is when I started to sweat, because the packet looked fine, my program was `XDP_TX` but the packet was nowhere ¯\_(ツ)_/¯.

## Down the rabbit hole

After realizing there was nothing wrong with the packet itself, I stopped and tried a fresh start: how to debug XDP?

Googling around, I found that ethtool is able to show some statistics about XDP, and that is what I did:

```bash
[root@3e7cf3b3e84c /]# ethtool -S eth0
NIC statistics:
     peer_ifindex: 15
     rx_queue_0_xdp_packets: 64
     rx_queue_0_xdp_bytes: 7944
     rx_queue_0_drops: 0
     rx_queue_0_xdp_redirect: 0
     rx_queue_0_xdp_drops: 0
     rx_queue_0_xdp_tx: 0
     rx_queue_0_xdp_tx_errors: 4

```

The statistics are showing tx_errors, but where can I find those errors?

A bit more googling lead me to learn how to check XDP tracepoints

### XDP Tracepoints

The [xdp tutorial](https://github.com/xdp-project/xdp-tutorial/blob/master/tracing02-xdp-monitor/README.org#tutorial-tracing02---monitor-xdp-tracepoints) contains a convenient way to monitor all the tracepoints.

[bpftrace](https://github.com/xdp-project/xdp-tutorial/blob/master/tracing02-xdp-monitor/README.org#bpftrace) seems
the more convenient way to monitor the tracepoints: it allows to monitor all the xdp* tracepoints and see the outcome:

```bash
sudo bpftrace -e 'tracepoint:xdp:* { @cnt[probe] = count(); }'
Attaching 12 probes...
^C

@cnt[tracepoint:xdp:xdp_bulk_tx]: 
```

There is only one tracepoint, xdp_bulk_tx so it must be it. As per the tutorial linked above, we can also
check for eventual errors:

```bash
bpftrace -e \
 'tracepoint:xdp:xdp_bulk_tx{@redir_errno[-args->err] = count();}'

Attaching 1 probe...
^C

@redir_errno[6]: 2
```

So, now we know the error code is -6! Brilliant! But... where that -6 comes from?

**It's time to look under the hood**: the tracepoint name is xdp_bulk_tx, which means that [somewhere in
the kernel](https://github.com/torvalds/linux/blob/0bb80ecc33a8fb5a682236443c1e740d5c917d1d/drivers/net/veth.c#L567) there is a
`trace_xdp_bulk_tx(rq->dev, sent, drops, err);` invocation that reports the error.

In my case it was clear that the `-6` I was getting was related to `-ENXIO` (here)[https://github.com/torvalds/linux/blob/0bb80ecc33a8fb5a682236443c1e740d5c917d1d/drivers/net/veth.c#L486], which is doc-commented as `/* No such device or address */`. 

I lack too much context in order to understand the particular scenario in which that error was returned, but [this comment](https://github.com/torvalds/linux/blob/0bb80ecc33a8fb5a682236443c1e740d5c917d1d/drivers/net/veth.c#L501) revealed itself to be another breadcrumb to the truth:

```C
	/* The napi pointer is set if NAPI is enabled, which ensures that
	 * xdp_ring is initialized on receive side and the peer device is up.
	 */
	if (!rcu_access_pointer(rq->napi))
		goto out;
```

This got me think that in case of veth, the other side must be XDP aware. I started googling and fell on this
**Veth pair swallow packets for XDP_TX operation** [thread](https://www.spinics.net/lists/netdev/msg625217.html)!

> You need to load a dummy program to receive packets from peer XDP_TX when using native veth XDP.
> 

So my "No such device or address" was related to the fact that there was no XDP program registered to the other side
of the veth pair, and I did not have any way to set that because the other end was handled by Docker itself.

## How I made it work

Given the issue was related mainly to the native veth implementation, what I did was just to replace the veth based bridge network with Macvlan based one, and it worked!

## Wrapping Up

I hope this will serve as an example of what tools are available to triage the obscure errors we can incur into when dealing with eBPF,
which sometimes are different from the workflow we (as regular kernel users) are used to follow.

Here I had to use eBPF to debug eBPF (via some tools like `bpftrace` and `xdpdump`) and eventually roll up my sleeves and dig into
the kernel sources to understand what that misterious error code I was getting was about.