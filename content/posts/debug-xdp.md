---
title: "Debug Dxp"
date: 2023-09-06T23:26:17+02:00
draft: true
---


### Let's start with the basics

I was not sure that the paramters I was passing to the program were correct. In fact, they weren't.

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
            "daddr": 175103499,
            "saddr": 175103243,
            "vip": 3232238081
        }
    }
]
```

This allowed me to ensure that the parameters I was passing were correct.

### Let's start with TCPDump

The swiss knife that saved me in multiple adventures is USELESS. If the packet is swallowed by XDP (as it happened to me), tcpdump won't
show anything. In my case, I was seeing only the SYN packet going from the `gateway` to the `katran` container and disappearing.

### XDPDump to the rescue

That's when I got to know [xdpdump](https://github.com/xdp-project/xdp-tools/tree/master/xdp-dump). XDP dump is a bit like TCPDump, but it
is able to inspect the packets going through (ingressing and egressing) an XDP program. So, by running

xdpdump 