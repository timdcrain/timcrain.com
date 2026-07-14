+++
title = "eBPF socket redirection"
date = 2026-07-12T21:43-07:00

[taxonomies]
tags = ["ebpf", "networking"]

[extra]
tagline = "Moving data between sockets . . . but with eBPF."
+++

I've recently been looking into improving performance for a proxy application.
The exact details of the proxy aren't too important, but once it finishes some
initial setup on a new connection the only thing it does is move data from one
TCP socket to another. This can be fairly expensive since each send and receive
results in two syscalls and copies—just to move some data! Luckily, this is the
sort of functionality—simple, no business logic—that is a perfect candidate to
move into the kernel with eBPF.

eBPF is a Linux[^linux] mechanism that allows verified[^verified] code to run
inside the kernel. If you've been paying attention to Linux since about
[2015][], you've most likely heard of it. While it started life as a packet
filter (the PF in eBPF), it has been extended to a broad range of networking and
non-networking use cases.

One eBPF feature is the ability to redirect data between sockets using socket
maps. For an application like the proxy I mentioned that primarily copies data
from one socket to another, this allows the kernel to move the data directly to
the target socket without any syscalls or copies. Theoretically, this should
reduce CPU utilization and latency and improve overall
performance.[^performance]

I want to go into more detail on how to use this feature in an actual
application, what problems you might run into, and whether the results actually
deliver on the promise.

But first, let's talk about the eBPF concepts that we're going to use.

## Socket maps

Socket maps are eBPF maps that hold references to sockets.[^socket-refs] There
are two types of socket maps: [`SOCKMAP`][], an array-backed map that uses a
32-bit lookup key, and [`SOCKHASH`][], a hash table that can use an arbitrary
key. `SOCKMAP` is less useful since it can't use arbitrary keys, so we're going
to be focusing on `SOCKHASH`.

Here's an example `SOCKHASH` map:

```C
struct socket_key {
    __u32 src_ip;
    __u32 dst_ip;
    __u32 src_port;
    __u32 dst_port;
};

struct {
    __uint(type, BPF_MAP_TYPE_SOCKHASH);
    __uint(max_entries, 1024);
    __type(key, struct socket_key);
    __type(value, __u64);
} sock_hash SEC(".maps");
```

The map is created with a fixed number of entries[^entries]—it doesn't
dynamically resize like hash maps in other languages. The key can be any
fixed-size type that can be represented as a flat sequence of bytes—i.e., no
pointers. In this example we're using a 4-tuple, but an eBPF program operating
on Layer 7 messages could use an application-level identifier instead. And
finally the value is either a 32- or 64-bit unsigned integer—the choice doesn't
matter too much because the sockets are put in the map either way.[^value]

Sockets can be added to the socket map from userspace with
`bpf_map_update_elem`.[^map-update] This will associate the socket with any eBPF
program attached to the socket map. Note that while a socket can be in multiple
socket maps, only one of those maps can have an attached eBPF program. If
removed from the socket map (via `bpf_map_delete_elem`), the socket will no
longer be associated with any eBPF programs and will function as a normal
socket.

## Verdict programs

A socket map by itself isn't all that useful, though. If we want to actually do
something with the sockets in the map, we have to attach a verdict program. Once
attached, the verdict program can choose to redirect, filter, or do nothing and
let the original socket process the data.

There are two types of verdict programs: [`sk_skb`][][^skb-verdict] and
[`sk_msg`][]. There are some important differences between the two program
types, which we'll discuss below, but let's look at an example of each first:

```C
SEC("sk_skb")
int bpf_skb_verdict_prog(struct __sk_buff *skb)
{
    // Only redirect if the dst port is 8080
    if (skb->local_port != 8080)
        return SK_PASS;

    struct socket_key key = {
        .src_ip = skb->remote_ip4,
        .dst_ip = skb->local_ip4,
        .src_port = skb->remote_port >> 16,
        // local_port is in host byte order
        .dst_port = (bpf_htonl(skb->local_port)) >> 16,
    };

    return bpf_sk_redirect_hash(skb, &sock_hash, &key, BPF_F_INGRESS);
}

SEC("sk_msg")
int bpf_msg_verdict_prog(struct sk_msg_md *msg)
{
    // Only redirect if the src port is 8080
    if (msg->local_port != 8080)
        return SK_PASS;

    struct socket_key key = {
        .src_ip = msg->local_ip4,
        .dst_ip = msg->remote_ip4,
        // local_port is in host byte order
        .src_port = (bpf_htonl(msg->local_port)) >> 16,
        .dst_port = msg->remote_port >> 16,
    };

    return bpf_msg_redirect_hash(msg, &sock_hash, &key, 0);
}
```

The programs are very similar, and in fact all of the differences are due to
direction—the `sk_skb` program is invoked when a socket receives data while the
`sk_msg` program is invoked when sending data to a socket.

Each program accepts what eBPF calls the program context. Both `__sk_buff` and
`sk_msg_md` are (mostly) read-only views of the corresponding kernel structs
`sk_buff` and `sk_msg`. These structs exist because the kernel doesn't provide a
stable ABI for internal structs. They also ensure that the verdict program can
only make safe mutations.

Both programs are redirecting connections on local port 8080. Connections on
other ports are untouched, and the kernel will pass them on to their original
socket. The 4-tuple from the input is used as the lookup key, with only the
source and destination differing to reflect the direction.

The redirect is done with the eBPF helper functions [`bpf_sk_redirect_hash`][]
and [`bpf_msg_redirect_hash`][].[^redirect-map] The helper looks up the key in
the socket map[^redirect-target] and, if the lookup succeeds, stores the socket
reference in a field on the corresponding kernel struct. The helper function
then returns `SK_PASS` to tell the kernel to continue processing the data. If
the lookup fails, the helper returns `SK_DROP` and the data will
(unsurprisingly) be dropped.[^sk-drop]

The redirect can either be an ingress redirect (using the `BPF_F_INGRESS` flag),
in which case the data will be put into the target socket's receive queue, or an
egress redirect (no flag), in which case the data will be put into the send
queue.

Besides the direction each verdict program operates on, the other major
difference is which socket types they support. `sk_skb` supports all socket
types (TCP, UDP, UDS, and VSOCK) and can redirect to any socket
type.[^vsock-skb] `sk_msg` on the other hand only supports TCP sockets and can
redirect to any socket type on ingress redirects,[^vsock-msg] but only TCP
sockets for egress redirects.

## Conclusion

There's a lot more that goes into making an eBPF program work. If you want to
learn more I highly recommend checking out [this talk][] by one of the
maintainers of socket maps, which goes into a lot more detail. In a future post
I'll compare proxies written in Rust and eBPF and find out if eBPF delivers the
potential performance improvements.

[2015]: https://en.wikipedia.org/wiki/EBPF
[`SOCKMAP`]: https://docs.ebpf.io/linux/map-type/BPF_MAP_TYPE_SOCKMAP/
[`SOCKHASH`]: https://docs.ebpf.io/linux/map-type/BPF_MAP_TYPE_SOCKHASH/
[`sk_skb`]: https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_SK_SKB/
[`sk_msg`]: https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_SK_MSG/
[this talk]: https://youtu.be/BMsfIu-UywY
[you might be right]: https://blog.cloudflare.com/sockmap-tcp-splicing-of-the-future/
[`sockops`]: https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_SOCK_OPS/
[`bpf_sk_redirect_hash`]: https://docs.ebpf.io/linux/helper-function/bpf_sk_redirect_hash/
[`bpf_msg_redirect_hash`]: https://docs.ebpf.io/linux/helper-function/bpf_msg_redirect_hash/
[`bpf_sk_redirect_map`]: https://docs.ebpf.io/linux/helper-function/bpf_sk_redirect_map/
[`bpf_msg_redirect_map`]: https://docs.ebpf.io/linux/helper-function/bpf_msg_redirect_map/

[^linux]: Ok ok, Windows too.
[^verified]: Verified in the sense that the code is statically analyzed for
    certain patterns (like infinite loops), not that it is signed or anything
    like that.
[^performance]: If that sounds too good to be true, [you might be right][]!
[^socket-refs]: While most examples (guilty) use TCP or UDP sockets, socket maps
    can hold any socket type. There are restrictions on the types of sockets
    that verdict programs can use, however, which we'll talk about later.
[^entries]: The hash map creates `max_entries` (rounded up to the nearest power
    of 2) buckets, and each bucket consists of a linked-list pointer and a
    spinlock for a 16-byte struct (on x86-64). The total size of all buckets
    must be less than or equal to 2<sup>32</sup> - 1 bytes, so `max_entries` can
    be at most the power of 2 below <sup>2<sup>32</sup> - 1</sup>⁄<sub>16</sub>,
    or 2<sup>27</sup>.
[^value]: Unless you want to look up an element in the map from userspace, in
    which case you should use the 64-bit value. The lookup will then return a
    unique identifier for the socket called a socket cookie.
[^map-update]: Sockets can also be added to the map from another eBPF program
    type called [`sockops`][], which is attached to cGroups and is triggered on
    TCP connection lifecycle events. This is useful when your application
    doesn't own the socket you want to redirect.
[^skb-verdict]: There are two different types of `sk_skb` verdict programs:
    `sk_skb/stream_verdict` and `sk_skb/verdict`. While both can function the
    same way, the former is TCP-only. `sk_skb/verdict` is a more recent
    (introduced in Linux 5.13) replacement that supports other socket types.
    The main reason to use `stream_verdict` is if the verdict needs to be made
    at Layer 7 (e.g., HTTP path-based routing), since `stream_verdict` can work
    in tandem with `sk_skb/stream_parser` which can determine Layer 7 message
    boundaries.
[^redirect-map]: For programs attached to `SOCKMAP` socket maps,
    [`bpf_sk_redirect_map`][] and [`bpf_msg_redirect_map`][] are used instead.
[^redirect-target]: Note that the redirect socket does not need to be in the
    same socket map as the one that the verdict program is attached to. This can
    be useful for mixing verdict programs, e.g., using an `sk_skb` program to
    redirect ingress data to a local socket and an `sk_msg` program to redirect
    data sent from that program to an egress socket.
[^sk-drop]: The verdict program can always ignore the return code of the helper
    function and return `SK_PASS` if the lookup fails. In that case the kernel
    will continue processing the data on the original socket.
[^vsock-skb]: Except for VSOCK on ingress redirects.
[^vsock-msg]: Again, except for VSOCK.
