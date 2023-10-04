+++ 
draft = false
date = 2023-09-24T14:59:22+02:00
title = "eBPF journey by examples: perf events with Pyroscope"
description = "Here we dig into pyroscope to understand how eBPF can fetch perf events and build indicators of how the application is behaving, providing a nice backend for flamegraphs."
slug = ""
authors = []
tags = []
categorie = ["Go", "ebpf", "performance"]
externalLink = ""
series = []
+++

# Pyroscope

After diving into [security with Falco]({{< ref "/posts/ebpf-journey-syscalls-with-falco" >}}) and [networking with XDP]({{< ref "/posts/ebpf-journey-loadbalancing-with-katran" >}}), I wanted to explore a profiling application that leverages eBPF.

**[Grafana Pyroscope](https://grafana.com/oss/pyroscope/) is an open source continuous profiling platform**. It supports various backends, including Go's `pprof`,
python via `py-spy` and many other languages. But the reason why I am writing this post is because it supports also an `eBPF`
backend.

## In order to profile, we need to understand what our program is doing

I don't know much about `py-spy`, but I know how Go's pprof works and it's very simple: it takes a snapshot of the stack
of every goroutine of the program over time. By aggregating such information we can understand where our program is
consuming most of the CPU: if the same stack is captured over time, it means it belongs to a time consuming path.

A common way to aggregate such information is via a [flamegraph](https://www.brendangregg.com/flamegraphs.html):

- The flamegraph does not represent the functions called over time, but all the functions captured in the sampling time
- Each level of the graph represents a level in the stack, with functions being called
- The width of each box represents how often that function appears in the dump (i.e. how long our program is busy in that function)


This for example is a flamegraph generated out of a MetalLB's speaker execution via the embedded Go pprof endpoint:

![](/images/pyroscope/torch.svg)


## With eBPF

Pyroscope uses both native profiling (for interpreted languages such as python, for example) but also eBPF based one, which is
the focus of this post.

eBPF provides a nice (and generic!) way to inspect the user or the kernel stack of a given application, via the `bpf_get_stackid`
eBPF helper.

From the bpf-helper man page:

> Walk a user or a kernel stack and return its id. To
> achieve this, the helper needs ctx, which is a
> pointer to the context on which the tracing program
> is executed, and a pointer to a map of type
> BPF_MAP_TYPE_STACK_TRACE.

So, the helper does two things:

- It walks the kernel (or user) stack and fills an element of the provided map of type `BPF_MAP_TYPE_STACK_TRACE`
- It generates and returns the key corresponding to that stack to access the map

This is just what we need to build our flamegraph: the eBPF program is notified when the sampling interval occurs and is
able to both count the occurrences of a given stack and to collect the stack.

The `pyroscope` eBPF side is pretty simple and self contained:

```C
SEC("perf_event")
int do_perf_event(struct bpf_perf_event_data *ctx)
{
    u64 id = bpf_get_current_pid_tgid();
    u32 tgid = id >> 32;
    u32 pid = id;
    struct sample_key key = { .pid = tgid};
    key.kern_stack = -1;
    key.user_stack = -1;
    u32 *val, one = 1, zero = 0;
    struct bss_arg *arg = bpf_map_lookup_elem(&args, &zero); // 1
    if (!arg) {
        return 0;
    }
    if (pid == 0) {
        return 0;
    }
    if (arg->tgid_filter != 0 && tgid != arg->tgid_filter) {
        return 0;
    }

    bpf_get_current_comm(&key.comm, sizeof(key.comm)); // 2

    if (arg->collect_kernel) {
        key.kern_stack = bpf_get_stackid(ctx, &stacks, KERN_STACKID_FLAGS); // 3
    }
    if (arg->collect_user)  {
        key.user_stack = bpf_get_stackid(ctx, &stacks, USER_STACKID_FLAGS); // 3 
    }

    val = bpf_map_lookup_elem(&counts, &key); // 4
    if (val)
        (*val)++;
    else
        bpf_map_update_elem(&counts, &key, &one, BPF_NOEXIST); // 4
    return 0;
}

```

What the program does is:

- it fetches the configuration (i.e. if filtering on the thread id must be performed) (1)
- it fetches the current command (2)
- depending on the configuration, fetches either the kernel stack or the user stack (3)
- it builds a key based on the command, the stack id and the thread id, and uses it to count how many
time a given stack is occurring (4)

The `stacks` map holds each different stack.

## On the userspace side

The userspace part is probably the most complex and interesting one. After registering the 
program, it translates the array of instruction pointers to the real stack.

### Attaching the program

For each CPU it creates the perf event:

```go
func newPerfEvent(cpu int, sampleRate int) (*perfEvent, error) {
    var (
       fd  int
       err error
    )
    attr := unix.PerfEventAttr{
       Type:   unix.PERF_TYPE_SOFTWARE,
       Config: unix.PERF_COUNT_SW_CPU_CLOCK,
       Bits:   unix.PerfBitFreq,
       Sample: uint64(sampleRate),
    }
    fd, err = unix.PerfEventOpen(&attr, -1, cpu, -1, unix.PERF_FLAG_FD_CLOEXEC)
    if err != nil {
       return nil, fmt.Errorf("open perf event: %w", err)
    }
    return &perfEvent{fd: fd}, nil
}
```

and then attaches the eBPF program to it:

```go
       err = pe.attachPerfEvent(s.bpf.profilePrograms.DoPerfEvent)
```

## Building the flame graph

In order to build the flame graph it collects all the keys of all the stacks, reading the counter map

```go
    keys, values, batch, err := s.getCountsMapValues()
    if err != nil {
       return fmt.Errorf("get counts map: %w", err)
    }
```

For each key it accesses the stacks map and builds a record with the pid, the stack, the counter:

```go
    for i := range keys {
      /*...*/
       if s.options.CollectUser {
         uStack = s.getStack(ck.UserStack)
       }
       if s.options.CollectKernel {
         kStack = s.getStack(ck.KernStack)
       }
       sfs = append(sfs, sf{
         pid:    ck.Pid,
         uStack: uStack,
         kStack: kStack,
         count:  value,
         comm:   getComm(ck),
         labels: labels,
       })
    }
```

Those `getStack` calls return the stack under the form of instruction pointers (as integers).

For each element, the code then _walks_ the stack of the corresponding pid, translates it to a readable form with the
name of the symbols and returns it via a callback:

```go
for _, it := range sfs {
       stats := stackResolveStats{}
       sb.rest()
       sb.append(it.comm)
       if s.options.CollectUser {
         s.walkStack(&sb, it.uStack, it.pid, &stats)
       }
       if s.options.CollectKernel {
         s.walkStack(&sb, it.kStack, 0, &stats)
       }
       if len(sb.stack) == 1 {
         continue // only comm
       }
       lo.Reverse(sb.stack)
       cb(it.labels, sb.stack, uint64(it.count), it.pid)
       s.debugDump(it, stats, sb)
    }
```

`walkStack` is where the magic happens:

```go
func (s *session) walkStack(sb *stackBuilder, stack []byte, pid uint32, stats *stackResolveStats) {
    if len(stack) == 0 {
       return
    }
    var stackFrames []string
    for i := 0; i < 127; i++ {
       instructionPointerBytes := stack[i*8 : i*8+8]
       instructionPointer := binary.LittleEndian.Uint64(instructionPointerBytes)
       if instructionPointer == 0 {
         break
       }
       sym := s.symCache.Resolve(pid, instructionPointer)
       var name string
       if sym.Name != "" {
         name = sym.Name
         stats.known++
       } else {
         if sym.Module != "" {
          //name = fmt.Sprintf("%s+%x", sym.Module, sym.Start) // todo expose an option to enable this
          name = sym.Module
          stats.unknownSymbols++
         } else {
          name = "[unknown]"
          stats.unknownModules++
         }
       }
       stackFrames = append(stackFrames, name)
    }

```

In order to be able to perform the translation, `pyroscope` fills this `symCache` object.

It reads the content of `/proc/$pid/maps` which gives the details of each
address region of the process' address space to a data structure under the form:

```go
    ProcMap{
       StartAddr: saddr,
       EndAddr:   eaddr,
       Perms:     perms,
       Offset:    offset,
       Dev:       device,
       Inode:     inode,
       Pathname:  pathname,
    }
```

Given the file corresponding to a given region, it uses `elf.NewFile()` to parse it.
The logic is then quite complex, so I will just provide the references on how pyroscope
fills the `symCache`:

- looking all the ranges to find where our instruction pointer belongs to [here](https://github.com/grafana/pyroscope/blob/07c51d833e73450c77800545ca0a3003d49fd049/ebpf/symtab/proc.go#L133)
- building an elf table using the file that corresponds to the given range [here](https://github.com/grafana/pyroscope/blob/07c51d833e73450c77800545ca0a3003d49fd049/ebpf/symtab/proc.go#L121)
- getting the elf file's `buildID` [here](https://github.com/grafana/pyroscope/blob/5acd4f8255ef154370e19bc8523aacf6531ad759/ebpf/symtab/elf.go#L98)
- trying to find the corresponding debug file to use it for the symbols [here](https://github.com/grafana/pyroscope/blob/5acd4f8255ef154370e19bc8523aacf6531ad759/ebpf/symtab/elf.go#L124)
- fetching the symbols from the elf table [here](https://github.com/grafana/pyroscope/blob/5acd4f8255ef154370e19bc8523aacf6531ad759/ebpf/symtab/elf.go#L156)

What we get at the end, is a list of stacks with their occurrences that can be stored somewhere and
then presented to the user as a nice flamegraph.

## Poor's man version

As always, I tried to write a simple version using Go for the userspace and C for the eBPF side,
stealing pieces from the project.

The simplified version still interacts with `perf_event`s, but it doesn't do the super-hard work
of translating the set of pointers to a human readable stack with names. It can be found in the usual
[github repo](https://github.com/fedepaol/ebpfexamples/tree/main/perfeventsample).

Let's have a look at the eBPF side:

```C
SEC("perf_event")
int do_perf_event(struct bpf_perf_event_data *ctx)
{
  u64 id = bpf_get_current_pid_tgid();
  u32 tgid = id >> 32;
  u32 pid = id;
  

  struct arguments *args = 0;
  __u32 argsKey = 0;
  args = (struct arguments *)bpf_map_lookup_elem(&params_array, &argsKey); // 1
  if (!args)
  {
    bpf_printk("no args");
    return -1;
  }

  if (pid != args->pid) {
    return 0; // 2
  }

  bpf_printk("got event for pid %d", pid);

  struct stack_key key;
  key.pid = pid;
  bpf_get_current_comm(&key.comm, sizeof(key.comm));
  key.stack_id = bpf_get_stackid(ctx, &stacks, USER_STACKID_FLAGS); // 3
  
  
  u32* val = bpf_map_lookup_elem(&counts, &key); // 4
  if (val)
    (*val)++;
  else {
    u32 one = 1;
    bpf_map_update_elem(&counts, &key, &one, BPF_NOEXIST);
  }

  return 0;
}
```

Here we:

- get the pid we are asked to observe (1)
- return if the pid we got the event for doesn't match (2)
- get the stack id (and contextually, fill the `stacks` map) (3)
- count and store how many times that stack appear (4)

On the userspace side we iterate over the CPUs, and set a perf event for
each one of them (heavily inspired by the pyroscope code):

```go
for _, cpu := range cpus {
        pe, err := newPerfEvent(int(cpu), 1)
        if err != nil {
            log.Fatalf("new perf event: %v", err)
        }
        opts := link.RawLinkOptions{
            Target:  pe.fd,
            Program: objs.DoPerfEvent,
            Attach:  ebpf.AttachPerfEvent,
        }

        pe.link, err = link.AttachRawLink(opts)
        if err != nil {
            log.Fatalf("attach raw link: %v", err)
        }
    }
```

Then we collect the data from the map containing the number of occurrences
for each different stack (still, of the same process we want to focus on):

```go
for i := 0; i < 10; i++ {
    time.Sleep(time.Second)
    var mapKey perfStackKey
    var count uint32
    iter := objs.Counts.Iterate() // 1

    for iter.Next(&mapKey, &count) {
        if count > maxCount {
            maxCount = count
            toDump = mapKey (2)
        }
    }
}
```

Here, for each round of polling we

- iterate over the `counts` map which contains the number of occurrences of any given stack (1)
- check if the count is the highest, and if so, we save the key (2)

The map key is something like:

```C
struct stack_key {
    __u32 pid;
    __s64 stack_id;
    char  comm[16];
};
```

So we can then fetch the actual stack using the stack_id part of the key, to find which is the stack that
appeared the most during our sampling (hence, were we consume the majority of our CPU):

```go
    stack, err := objs.Stacks.LookupBytes(uint32(toDump.StackId)) // 1
    if err != nil {
        log.Fatalf("stacks lookup: %v", err)
    }
    for i := 0; i < 127; i++ { // 2
        instructionPointerBytes := stack[i*8 : i*8+8]
        instructionPointer := binary.LittleEndian.Uint64(instructionPointerBytes) // 3
        if instructionPointer == 0 {
            break
        }
        fmt.Println("Instruction pointer", instructionPointer) // 4
    }
```

We:

- fetch the stack information as an array of bytes (1)
- iterate over the maximum depth (2)
- take a subslice containing the corresponding element of the stack (3)
- print the stack element out (4)

## Wrapping Up

This was *by far* the most complex project to dig into, not because of the complexity of the eBPF program,
but because of the super hard job it makes to convert those instruction pointers to something meaninful to the
end user.

After digging into security with Falco and networking with Katran, here we explored another side
of eBPF: observability.

As always, this is the result of my personal observations reading the code. Feel free to comment if something is
not accurate and might be presented in a better fashion.