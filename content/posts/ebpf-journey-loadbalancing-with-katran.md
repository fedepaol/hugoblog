---
title: "eBPF journey by examples: xdp redirecting with Katran"
date: 2023-08-21T15:01:51+01:00
categories: ["Go", "ebpf", "xdp"]
#description: Discover eBPF's power with tracepoints and KProbes through practical examples. Explore how Falco, a cloud-native security tool, leverages these techniques for Linux system monitoring. Learn to implement user space logic using Go and uncover the complexities of eBPF programming.
draft: true
---

# Katran

This is my second post about digging into popular eBPF projects. This time I am looking at [katran](https://github.com/facebookincubator/katran), an xdp based
loadbalancer opensourced by Meta.

A loadbalancer works in a very simple way: it translates network packets directed towards a single virtual IP (VIP) toward a serie of endpoints (`real`s, in katran's lingo),
while maintaining session consistency (meaning, all the packets belonging to a given tcp session are sent to the same `real`).

Katran's loadbalancing is peculiar in the sense that:

- it assumes all the `reals` are reacheable via a single next hop katran is configured with, which is expected to do the routing logic
- works in DSR mode (direct server return), meaning the reply is not getting back through the load balancer but will come directly from the server
- each real has the VIP associated to a loopback interface. This allows the real to reply to packets directed to the VIP
- the packet directed to the VIP is encapsulated via IPIP inside a packet directed to the real (this allows the reals to be on different subnets)

The flow looks like:

```none
                                        ┌───────┬───────┬┬──────┬───┬─────┬─┐
                                        │Katran │Real IP││Client│VIP│Data │ │
                                        │       │       ├┴──────┴───┴─────┘ │
                                        │       │       │                   │
              ┌──────┬───┬─────┐        └───────┴───────┴───────────────────┘
              │Client│VIP│Data │
┌───────────┐ └──────┴───┴─────┘  ┌───────────┐                  ┌───────────┐
│           │                     │           │                  │           │
│  Client   ├────────────────────►│  Katran   ├─────────────────►│  Real     │
│           │                     │           │                  │           │
└───────────┘                     └───────────┘                  └────┬──────┘
    ▲                                                                 │
    │                                                                 │
    │                                                                 │
    │                                                                 │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘

                            ┌───┬──────┬─────┐
                            │VIP│Client│Data │
                            └───┴──────┴─────┘
```

Where each packet is represented by src, dst and payload.

## XDP

By leveraging XDP, katran is able to:

- intercept incoming packets directed to the VIP
- calculate the IP of the `real` related to that session
- transform the packet to a packet directed to the `real` IP (even by changing the ip family)
- encapsulate the packet and send it back to the interface it came from

## Looking at the code

The `balancer_ingress` XDP program is the [entry point](https://github.com/facebookincubator/katran/blob/7cca2aae1607ab6770d80b08ec640b7f9dc5106f/katran/lib/bpf/balancer_kern.c#L934) for the balancing logic, which is then deferred to the `process_packet` function.

Leaving corner cases aside, what `process_packet` does is:

### finding the dst address corresponding to the packet directed to the `vip`

```C
    if (!dst && !(pckt.flags & F_SYN_SET) &&
        !(vip_info->flags & F_LRU_BYPASS)) {
      connection_table_lookup(&dst, &pckt, lru_map, /*isGlobalLru=*/false); // 1
    }

    if (!dst && !(pckt.flags & F_SYN_SET) && vip_info->flags & F_GLOBAL_LRU) { // 2
      int global_lru_lookup_result =
          perform_global_lru_lookup(&dst, &pckt, cpu_num, vip_info, is_ipv6);
      if (global_lru_lookup_result >= 0) {
        return global_lru_lookup_result;
      }
    }

    // if dst is not found, route via consistent-hashing of the flow.
    if (!dst) {
      /* ... stats handling ... */
      if (!get_packet_dst(&dst, &pckt, vip_info, is_ipv6, lru_map)) { // 3
        return XDP_DROP;
      }

      /* ... stats handling ... */
    }
  }
```

- if the packet is a SYN packet, the session is a new one and thus it does not make sense to look into the LRU cache
- if the packet is not a SYN packet, katran performs a lookup into a per-cpu lru cache (1), then into a global LRU cache (2)
- if the real IP is not found, then a hashing function is performed and the dst is calculated (3)

One interesting thing to notice is the set of properties related to the vip (`vip_info->flags`), which allow to skip the caches
for example (the rationale behind this is to avoid filling up the memory).

### Getting the packet destination

The computation performed by `get_packet_dst` is stateless and depends on the packet. The gist of the way it (more or less) works is:

```C
    hash = get_packet_hash(pckt, hash_16bytes) % RING_SIZE; // 1
    key = RING_SIZE * (vip_info->vip_num) + hash; // 2

    real_pos = bpf_map_lookup_elem(&ch_rings, &key); // 3 
    if (!real_pos) {
      return false;
    }
    key = *real_pos;

    pckt->real_index = key;
    *real = bpf_map_lookup_elem(&reals, &key); // 4
```

- the hash of the packet is calculated and modulized RING_SIZE
- the key is RING_SIZE (an array containing all the real indexes for all the vips, ordered in chunks). The i-th VIP chunk is between 
`RING_SIZE * (vip_info->vip_num)`` and `RING_SIZE * (vip_info->vip_num) + RING_SIZE`. My guess is, this allows to have a non even distributions 
of reals (3)
- the real is finally fetched (4)

### Encapsulating and redirecting the packet

Once the dst is found, the redirection takes place:

```C
  cval = bpf_map_lookup_elem(&ctl_array, &mac_addr_pos); // 1

  if (!cval) {
    return XDP_DROP;
  }

  /* ... stats .. */
  if (dst->flags & F_IPV6) {
    if (!PCKT_ENCAP_V6(xdp, cval, is_ipv6, &pckt, dst, pkt_bytes)) {
      return XDP_DROP;
    }
  } else {
    if (!PCKT_ENCAP_V4(xdp, cval, &pckt, dst, pkt_bytes)) { // 2
      return XDP_DROP;
    }
  }

  return XDP_TX; // 3

```

- The details of the next hop are fetched (1)
- The new (encapsulated) packet is generated, with the `real` ip as dst, a crafted src ip and the original packet as payload (2)
- The XDP_TX verdict is returned, meaning that the packet is not passed to the kernel stack but sent out back to the same interface it was received (3)

I won't expand too much the details of the encapsulation, that can be found [here](https://github.com/facebookincubator/katran/blob/1f464b97d47750ce0195cf5b7789d7524d8a1110/katran/lib/bpf/pckt_encap.h#L95), as they involve manipulating the headers, setting the right payload, src and dst and recalculating the checksum.

It's interesting to notice that the encapsulation depends on the ip family of the real, and might be of a different family compared to the VIP.
Also, the tos flags are preserved.

## A hidden gem

While looking for all the eBPF programs in the repo, I found [this sockops program](https://github.com/facebookincubator/katran/blob/6cb4f69303a705a1a3b54bc2776611a85ae3099f/katran/tpr/bpf/tcp_pkt_router_kern.c#L109). After digging a bit, my understanding is that it allows the server to send the serverid to the client directly into the tcp header options
(in the syn/ack message):

```C
    case BPF_SOCK_OPS_WRITE_HDR_OPT_CB:
      /* Write the server-id as hdr-opt */
      if ((skops->skb_tcp_flags & TCPHDR_SYNACK) == TCPHDR_SYNACK) {
        return handle_passive_write_hdr_opt(skops, stat, s_info);
      } else {
        return SUCCESS;
      }
    /*...*/


    hdr_opt.kind = TCP_HDR_OPT_KIND;
    hdr_opt.len = TCP_HDR_OPT_LEN;
    hdr_opt.server_id = s_info->server_id;
    err = bpf_store_hdr_opt(skops, &hdr_opt, sizeof(hdr_opt), NO_FLAGS);
```

On the client side on the other hand, parses the serverid coming from the syn/ack message :

```C
      /* Read hdr-opt sent by the passive side
       * Only parse the SYNACK because server TPR only sends OPT with SYNACK */
      if ((skops->skb_tcp_flags & TCPHDR_SYNACK) == TCPHDR_SYNACK) {
        return handle_active_parse_hdr(skops, stat);
      } else {
        return SUCCESS;
      }
```

To set it back in sequent messages:

```C
    case BPF_SOCK_OPS_WRITE_HDR_OPT_CB:
      /* Echo back the server-id as hdr-opt
       * Don't attempt to do this for the SYN packet because we only
       * get the OPT in the SYNACK */
      if ((skops->skb_tcp_flags & TCPHDR_SYN) != TCPHDR_SYN) {
        return handle_active_write_hdr_opt(skops, stat);
      } else {
        return SUCCESS;
      }
```

This trick allows the XDP program to avoid even accessing the LRU maps / calculating the id of the real, because the id
is already contained in the tcp header, in the [following part](https://github.com/facebookincubator/katran/blob/7cca2aae1607ab6770d80b08ec640b7f9dc5106f/katran/lib/bpf/balancer_kern.c#L744) of the `process_packet` function I omitted commenting:

```C
  if (!dst) {
#ifdef TCP_SERVER_ID_ROUTING
    // First try to lookup dst in the tcp_hdr_opt (if enabled)
    if (pckt.flow.proto == IPPROTO_TCP && !(pckt.flags & F_SYN_SET)) {
      __u32 routing_stats_key = MAX_VIPS + TCP_SERVER_ID_ROUTE_STATS;
      struct lb_stats* routing_stats =
          bpf_map_lookup_elem(&stats, &routing_stats_key);
      if (!routing_stats) {
        return XDP_DROP;
      }
      if (tcp_hdr_opt_lookup(xdp, is_ipv6, &dst, &pckt) == FURTHER_PROCESSING) {
        routing_stats->v1 += 1;
      } else {
        // update this routing decision in the lru_map as well
        if (lru_map && !(vip_info->flags & F_LRU_BYPASS)) {
          check_and_update_real_index_in_lru(
              &pckt, lru_map, /* update_lru */ true);
        }
        routing_stats->v2 += 1;
      }
    }
#endif // TCP_SERVER_ID_ROUTING
```

### Poor man's version

Here I will try to mimic the very minimum of what Katran implements, avaliable at the [following link](https://github.com/fedepaol/ebpfexamples/tree/main/xdpkatransample).

The parameters are fixed, but fed via a BPF_MAP_TYPE_ARRAY to the program. A fancier and more complex application might have a map
depending on the VIP and other parameters. The parameters include:

- The mac address of the next hop
- The VIP to be translated
- The IP of the "real"

The program does the filtering of non IPV4 packets, and non TCP packets, and then leverages the encapsulation logic stolen from Katran:

It reads the arguments from the map provided by the Go side:

```C
  struct arguments *args = 0;
  __u32 key = 0;
  args = (struct arguments *)bpf_map_lookup_elem(&xdp_params_array, &key);
  if (!args)
  {
    bpf_printk("no args");
    return XDP_PASS;
  }
```

After that it prepares the encapsulated packet:

```C
  if (bpf_xdp_adjust_head(xdp, 0 - (int)sizeof(struct iphdr))) // 1
  {
    return false;
  }

  memcpy(new_eth->h_dest, dst_mac, 6); // 2
  if (old_eth->h_dest + 6 > data_end)
  {
    return false;
  }
  memcpy(new_eth->h_source, old_eth->h_dest, 6); // 3 
  new_eth->h_proto = BE_ETH_P_IP;

  create_v4_hdr(
    iph,
    saddr,
    daddr,
    pkt_bytes,
    IPPROTO_IPIP); // 4
```

It:

- Makes room for the header of the IPIP tunnel (1)
- Sets the source mac address using the original destination one, the one of the interface (2)
- Sets the destination mac address as the one provided by the parameters (the gateway one) (3)
- Creates the IPIP header using the destination of the real as the destination IP

## Testing it (for real)

The [repro subfolder](https://github.com/fedepaol/ebpfexamples/tree/main/xdpkatransample/repro) contains the setup I used
to validate my ramblings were correct.

I used a container-based environment using `docker-compose` (I know, it's not a-la-mode anymore) with multiple
networks, setting the right routes in the various containers.

The containers layout look a bit like:

```raw
              10.111.220.0                     10.111.222.0/24

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

Where the gateway container mimics the router, there are three different network segments:

- client - gateway
- katran - gateway
- client - real

On the `real` container we also setup an ipip interface as per the instructions on katran.

The setup files can be found in the [github repo](https://github.com/fedepaol/ebpfexamples/tree/main/xdpkatransample/repro/setup).

Another thing to note is that the RP_FILTER parameter must be disabled on the local machine (running 
`echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter` should make the trick).

## Profit!

Well, no. This is the reason why this second post took so long to be published. I spent endless nights trying to debug why it wasn't
working. However, I think it's worth sharing some of the tools that I discovered in the process.

### Let's start with TCPDump

The swiss knife that saved me in multiple adventures is USELESS. If the packet is swallowed by XDP (as it happened to me), tcpdump won't
show anything. In my case, I was seeing only the SYN packet going from the `gateway` to the `katran` container and disappearing.

### XDPDump to the rescue

That's when I got to know [xdpdump](https://github.com/xdp-project/xdp-tools/tree/master/xdp-dump). XDP dump is a bit like TCPDump, but it
is able to inspect the packets going through (ingressing and egressing) an XDP program. So, by running

xdpdump 