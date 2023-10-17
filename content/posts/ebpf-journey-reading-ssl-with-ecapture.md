+++
draft = false
date = 2023-10-13T15:41:18+02:00
title = "eBPF journey by examples: hijacking SSL with eBPF with eCapture"
description = "a post about how ecapture leverages eBPF uprobes and uretprobes in a very creative way, by intercepting and showing openssl traffic in clear"
slug = ""
authors = []
tags = ["go", "ebpf"]
categories = []
externalLink = ""
series = []
+++

# ECapture

Uprobes (user probes) and uretprobes are a way to attach an eBPF program to a user space application, specifically to the entry or exit point of a given function. By doing so, the program is able to get notified whenever a given function is called, and with which arguments. This information is often
used to notify the user space side of the eBPF program, that handles it to serve the purpouse of the application.

[ECapture](https://github.com/gojue/ecapture) provides a creative usage of eBPF: it leverages it to intercept ssl traffic before it actually gets encrypted, by using eBPF programs of type uprobe / uretprobe (mostly).

Despite the natural total lack of guarantee of stability with uprobes (even less than `kprobe`s!), related to the fact that after a refactor
a given function might go away, the exposed API of popular libraries have a better chance not to change across releases. `SSL_write` for example
is likely to remaining stable.

## Intercepting SSL calls

ECapture offers different operational modes, including attaching to a network interface and printing the output to a wireshark parseable file, but in this post I will focus on the part that intercepts `SSL_write` to fetch the various arguments the function is called with (including the buffer being written).

## The eBPF program

The `ssl_write` pre / post hook looks a lot like the kprobe programs I described when I looked under falco's hood [here]({{< ref "/posts/ebpf-journey-syscalls-with-falco" >}}).

We have a program for when we enter the `SSL_write` function and one for when we exit.

### The uprobe part

The uprobe program fetches the current pid / gid and applies a filter:

```C
SEC("uprobe/SSL_write")
int probe_entry_SSL_write(struct pt_regs* ctx) {
    u64 current_pid_tgid = bpf_get_current_pid_tgid();
    u32 pid = current_pid_tgid >> 32;
    u64 current_uid_gid = bpf_get_current_uid_gid();
    u32 uid = current_uid_gid;

#ifndef KERNEL_LESS_5_2
    // if target_ppid is 0 then we target all pids
    if (target_pid != 0 && target_pid != pid) {
        return 0;
    }
    if (target_uid != 0 && target_uid != uid) {
        return 0;
    }
#endif
```

#### Filtering the pid

I'll spend a second describing how that pid to filter is injected from outside. Instead of using a map of type `BPF_ARRAY`
as seen in other projects, here the pid value to match is a [global constant](https://github.com/gojue/ecapture/blob/d5a768122b8343354c7a22b45ff0a19b5ffb4877/kern/common.h#L54):

```C
// Optional Target PID and UID
const volatile u64 target_pid = 0;
const volatile u64 target_uid = 0;
```

The way eCapture sets it from userspace is via this [constanteditor](https://github.com/gojue/ecapture/blob/d5a768122b8343354c7a22b45ff0a19b5ffb4877/user/module/probe_openssl.go#L247) that hammers the required value directly
in the compiled eBPF program. It can obviously be done only before the program itself is loaded.

The code used to rewrite the uint64 value of the program can be found in the [ebpfmanager library](https://github.com/gojue/ebpfmanager/blob/9a1bbe7454b0433455fdf6e31178f3744506a5c2/editor.go#L46) used by ecapture.

### Fetching the ssl_write arguments

The ssl_write signature looks like: `int SSL_write(SSL *ssl, const void *buf, int num);`, and the program leverages
the `PT_REGS_PARMXX` macros to fetch the arguments directly from the context.

```C
    debug_bpf_printk("openssl uprobe/SSL_write pid :%d\n", pid);

    void* ssl = (void*)PT_REGS_PARM1(ctx);
    // https://github.com/openssl/openssl/blob/OpenSSL_1_1_1-stable/crypto/bio/bio_local.h
    struct ssl_st ssl_info;
    bpf_probe_read_user(&ssl_info, sizeof(ssl_info), ssl);

    struct BIO bio_w;
    bpf_probe_read_user(&bio_w, sizeof(bio_w), ssl_info.wbio);

    // get fd ssl->wbio->num
    u32 fd = bio_w.num;
    debug_bpf_printk("openssl uprobe SSL_write FD:%d\n", fd);

    const char* buf = (const char*)PT_REGS_PARM2(ctx);
}
```

### Storing the relevant information to be used in the uretprobe

This is a common pattern in the kprobe / kretprobe too. We save the parameters in a map having the pid / tgid as the key,
and the values we want to retrieve on exit as value (in this case the file descriptor, the buffer and the version).

```C
   struct active_ssl_buf active_ssl_buf_t;
    __builtin_memset(&active_ssl_buf_t, 0, sizeof(active_ssl_buf_t));
    active_ssl_buf_t.fd = fd;
    active_ssl_buf_t.version = ssl_info.version;
    active_ssl_buf_t.buf = buf;
    bpf_map_update_elem(&active_ssl_write_args_map, &current_pid_tgid,
                        &active_ssl_buf_t, BPF_ANY);

    return 0;
```

### The uretprobe part

The uretprobe is called when the `ssl_write` function returns.

```C
SEC("uretprobe/SSL_write")
int probe_ret_SSL_write(struct pt_regs* ctx) {
    u64 current_pid_tgid = bpf_get_current_pid_tgid(); // 1
    u32 pid = current_pid_tgid >> 32;
    u64 current_uid_gid = bpf_get_current_uid_gid();
    u32 uid = current_uid_gid;

#ifndef KERNEL_LESS_5_2
    // if target_ppid is 0 then we target all pids
    if (target_pid != 0 && target_pid != pid) { // 2
        return 0;
    }
    if (target_uid != 0 && target_uid != uid) {
        return 0;
    }
#endif
    debug_bpf_printk("openssl uretprobe/SSL_write pid :%d\n", pid);
    struct active_ssl_buf* active_ssl_buf_t = // 3
        bpf_map_lookup_elem(&active_ssl_write_args_map, &current_pid_tgid);
    if (active_ssl_buf_t != NULL) {
        const char* buf;
        u32 fd = active_ssl_buf_t->fd;
        s32 version = active_ssl_buf_t->version;
        bpf_probe_read(&buf, sizeof(const char*), &active_ssl_buf_t->buf);
        process_SSL_data(ctx, current_pid_tgid, kSSLWrite, buf, fd, version); // 4
    }
    bpf_map_delete_elem(&active_ssl_write_args_map, &current_pid_tgid);
    return 0;
}
```

The uretprobe program is quite simple:

- it retrieves the pid / tgid (1)
- filters the pid (if the filter is set) (2)
- retrieves the data stored in the uprobe (3)
- invokes `process_SSL_data` (4)
- deletes the element from the map (5)

Now, let's have a look at how the event is processed.

### Processing the ssl write data

The function leverages the `PT_REGS_RC` which allows to retrieve the return value of the function. If the `ssl_write` call failed (returning a value lower than 0)
the function exits:

```C
static int process_SSL_data(struct pt_regs* ctx, u64 id,
                            enum ssl_data_event_type type, const char* buf,
                            u32 fd, s32 version) {
    int len = (int)PT_REGS_RC(ctx);
    if (len < 0) {
        return 0;
    }
```

Otherwise it builds an event entry with the command, the file descriptor, the ssl version and the buffer:

```C
    struct ssl_data_event_t* event = create_ssl_data_event(id);
    if (event == NULL) {
        return 0;
    }

    event->type = type;
    event->fd = fd;
    event->version = version;
    // This is a max function, but it is written in such a way to keep older BPF
    // verifiers happy.
    event->data_len =
        (len < MAX_DATA_SIZE_OPENSSL ? (len & (MAX_DATA_SIZE_OPENSSL - 1))
                                     : MAX_DATA_SIZE_OPENSSL);
    bpf_probe_read_user(event->data, event->data_len, buf);
    bpf_get_current_comm(&event->comm, sizeof(event->comm));
    bpf_perf_event_output(ctx, &tls_events, BPF_F_CURRENT_CPU, event,
                          sizeof(struct ssl_data_event_t));
    return 0;
}
```

By setting the `BPF_F_CURRENT_CPU` the event is written to the map entry related to the current cpu index.
Another thing to note is how I already highlighted in the [post about falco]({{< ref "/posts/ebpf-journey-syscalls-with-falco" >}}), that
ringbuffers are a more performant alternatives over perf buffers.

### The userspace side

A goroutine [polls the perfbuffer](https://github.com/gojue/ecapture/blob/dbc827e0eeeb84c233a022063e952ae2f68616b1/user/module/imodule.go#L200),
decodes it and processes it:

```C
record, err := rd.Read()
if err != nil {
    if errors.Is(err, perf.ErrClosed) {
        return
    }
    errChan <- fmt.Errorf("%s\treading from perf event reader: %s", m.child.Name(), err)
    return
}

if record.LostSamples != 0 {
    m.logger.Printf("%s\tperf event ring buffer full, dropped %d samples", m.child.Name(), record.LostSamples)
    continue
}

var e event.IEventStruct
e, err = m.child.Decode(em, record.RawSample)
if err != nil {
    m.logger.Printf("%s\tm.child.decode error:%v", m.child.Name(), err)
    continue
}

// 上报数据
m.Dispatcher(e)
```

#### Decoding and processing the event

Decoding just goes over the payload filling the structure. Code [here](https://github.com/gojue/ecapture/blob/dbc827e0eeeb84c233a022063e952ae2f68616b1/user/event/event_openssl.go#L84)

```go
func (se *SSLDataEvent) Decode(payload []byte) (err error) {
    buf := bytes.NewBuffer(payload)
    if err = binary.Read(buf, binary.LittleEndian, &se.DataType); err != nil {
        return
    }
    if err = binary.Read(buf, binary.LittleEndian, &se.Timestamp); err != nil {
        return
    }
    /* other fields */

    return nil
}
```

Processing the event just calls the `String()`ed version of the event, which at the end of the day [prints the following](https://github.com/gojue/ecapture/blob/dbc827e0eeeb84c233a022063e952ae2f68616b1/user/event/event_openssl.go#L152):

```go
    s := fmt.Sprintf("PID:%d, Comm:%s, TID:%d, Version:%s, %s, Payload:\n%s%s%s", se.Pid, bytes.TrimSpace(se.Comm[:]), se.Tid, v.String(), connInfo, perfix, string(se.Data[:se.DataLen]), COLORRESET)
```

And that's it! The user expects the traffic to be encrypted, but here we intercept it after the client side called `SSL_write` but before it
gets encrypted.

## Poor man's version

As per the other projects, I tried to reproduce the feature in a simplified fashion. The code can be found [here](https://github.com/fedepaol/ebpfexamples/tree/main/sslwriteuprobe).

The ebpf programs are basically the same I showed up above, with a few tiny differences:

- we store only the buffer in the pre -> post map
- we use a ring buffer to send the data back to userspace

```C
SEC("uprobe/SSL_write")
int probe_entry_SSL_write(struct pt_regs *ctx)
{
    u64 current_pid_tgid = bpf_get_current_pid_tgid();
    u32 pid = current_pid_tgid >> 32;

    struct arguments *args = 0;
    __u32 argsKey = 0;
    args = (struct arguments *)bpf_map_lookup_elem(&params_array, &argsKey);
    if (!args)
    {
        bpf_printk("no args");
        return -1;
    }
    if (args->pid != 0 && args->pid != pid)
    {
        return 0;
    }

    bpf_printk("openssl uprobe/SSL_write pid :%d\n", pid);

    const char *buf = (const char *)PT_REGS_PARM2(ctx);
    struct active_ssl_buf active_ssl_buf_t;
    active_ssl_buf_t.buf = (uintptr_t)buf;
    bpf_map_update_elem(&active_ssl_write_args_map, &current_pid_tgid,
                        &active_ssl_buf_t, BPF_ANY);

    return 0;
}
```

The pid to filter is taken from a map (of type ARRAY), and we only save the buffer to be retrieved by the uret probe call:

```C
SEC("uretprobe/SSL_write")
int probe_ret_SSL_write(struct pt_regs *ctx)
{
    /**
    filter by pid
    **/
    struct active_ssl_buf *active_ssl_buf_t =
        bpf_map_lookup_elem(&active_ssl_write_args_map, &current_pid_tgid);
    if (active_ssl_buf_t != NULL)
    {
        const char *buf;
        bpf_probe_read(&buf, sizeof(const char *), &active_ssl_buf_t->buf);
        process_SSL_data(ctx, current_pid_tgid, buf);
    }
    bpf_map_delete_elem(&active_ssl_write_args_map, &current_pid_tgid);
    return 0;
}
```

On the uretprobe side we fetch the entry of the map related to the current execution, we build the event to send
and then we eventually send it:

```C
static int process_SSL_data(struct pt_regs *ctx, u64 id,
                            const char *buf)
{
    int len = (int)PT_REGS_RC(ctx);
    if (len < 0)
    {
        return 0;
    }

    bpf_printk("openssl process_SSL_data len :%d buf %s\n", len, buf);

    struct ssl_data_event_t *event = 0;
    event = bpf_ringbuf_reserve(&ring_buffer, sizeof(struct ssl_data_event_t), 0);
    if (!event)
    {
        return 0;
    }

    event->timestamp_ns = bpf_ktime_get_ns();
    event->pid = id >> 32;

    event->data_len =
        (len < MAX_DATA_SIZE_OPENSSL ? (len & (MAX_DATA_SIZE_OPENSSL - 1))
                                     : MAX_DATA_SIZE_OPENSSL);
    bpf_probe_read_user(event->data, event->data_len, buf);
    bpf_get_current_comm(&event->comm, sizeof(event->comm));

    bpf_ringbuf_submit(event, 0);
    return 0;
}
```

On the user space side we just register the two programs and loop over the ringbuffer. The simplified version
looks like:

```go
up, _ := ex.Uprobe("SSL_write", objs.ProbeEntrySSL_write, nil)
defer up.Close()
up1, _ := ex.Uretprobe("SSL_write", objs.ProbeRetSSL_write, nil)
defer up1.Close()
rd, _ := ringbuf.NewReader(objs.uprobeMaps.RingBuffer)
```

we then iterate over the ring buffer and print the value.

## Wrapping up

Despite the structure being simple (or me getting used to how the verifier thinks and what needs to be done to have a working eBPF program), I really
enjoyed doing this because of the creative use case of uprobes to steal encrypted data (and seeing it in action!).

I hope this can serve as the base for any experimentation around uprobes.
