---
title: "EBPF TC filters for egress traffic"
date: 2023-04-06T15:01:51+01:00
categories: ["Go", "ebpf", "networking"]
---

# Network filters with EBPF and TC

EBPF has been on my radar for a while, and given my involvement with MetalLB, starting from a network hook felt natural.

So, I started with a very simple idea in mind: writing a hook that drives the egress traffic to a different next hop, based on some criteria. This comes from a real problem
that a few MetalLB users raised: we have an asymmetric return path when the traffic comes to the node via a secondary (i.e. non default gateway) interface and the client is
multiple hops away (and we don't have a routing strategy on the host).

![](/images/tcegress/scheme.png)


One possible solution (also implemented by the [metallb node route agent](https://github.com/travisghansen/metallb-node-route-agent)) is to use strategies like source based routing (or marking based) to get the traffic exit via the right interface, but hey
EBPF is _a la mode_, so why not to try to use it?

## First hurdles

I really struggled to find the right documentation. XDP is well documented, it has an excellent tutorial which I tried to follow, and there's a nice example on how to
run an xdp hook from go.

But the problem is, XDP works only for ingress. 
On the other side, TC is well known to _control traffic_ with classes, queuing disciplines and filters, but it's relationship with ebpf wasn't totally clear to me.

The best source of information for how the tc / ebpf machinery work is [this blogpost by Quentin Monnet](https://qmonnet.github.io/whirl-offload/2020/04/11/tc-bpf-direct-action/), where he demistifies the `direct-action` and the `clsact` qdiscs and how they allow the action itself to be embedded in the filters
implemented as ebpf programs, instead of having the action listed in the filter declaration.

## Just give me some code to copy!

This is by far the biggest issue I had. I wasn't able to find a proper example on how to create such filter and how to load it from Go, and such example was
lacking from the [cilium ebpf library examples](https://github.com/cilium/ebpf/tree/master/examples).

Anyway, after digging into issues and discussions [1](https://github.com/cilium/ebpf/issues/768),[2](https://github.com/cilium/ebpf/discussions/769), [3](https://github.com/florianl/tc-skeleton/discussions/2), and also by ~~ruthless copying~~ taking inspiration from the cilium codebase, I was able to put together the right enchantment
to load an ebpf filter via TC.


```go
	devID, err := net.InterfaceByName("eth0")
	if err != nil {
		return fmt.Errorf("could not get interface ID: %w", err)
	}

	qdisc := &netlink.GenericQdisc{
		QdiscAttrs: netlink.QdiscAttrs{
			LinkIndex: devID.Index,
			Handle:    netlink.MakeHandle(0xffff, 0),
			Parent:    netlink.HANDLE_CLSACT,
		},
		QdiscType: "clsact",
	}

	err = netlink.QdiscReplace(qdisc)
	if err != nil {
		return fmt.Errorf("could not get replace qdisc: %w", err)
	}

	filter := &netlink.BpfFilter{
		FilterAttrs: netlink.FilterAttrs{
			LinkIndex: devID.Index,
			Parent:    netlink.HANDLE_MIN_EGRESS,
			Handle:    1,
			Protocol:  unix.ETH_P_ALL,
		},
		Fd:           program.FD(),
		Name:         program.String(),
		DirectAction: true,
	}

	if err := netlink.FilterReplace(filter); err != nil {
		return fmt.Errorf("failed to replace tc filter: %w", err)
	}
```

Both the `handle` and the `parent` ids passed to the qdisc and to the filter are taken with a leap of faith from the tc implementation itself [1](https://github.com/shemminger/iproute2/blob/main/tc/tc_qdisc.c#L92), [2](https://github.com/shemminger/iproute2/blob/main/tc/tc_filter.c#L118).


The full example including a docker-compose for validation is available on github at [github.com/fedepaol/tc-return](https://github.com/fedepaol/tc-return).

## Conclusion

I hope this blogpost will help people looking on how to use EBPF to steer egress traffic via tc to save some time. Ebpf is a powerful tool, but sometimes the information
on how to use it properly is scattered across different sources.

