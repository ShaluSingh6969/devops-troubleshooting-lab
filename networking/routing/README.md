# Linux Routing, Subnetting, and Neighbor Discovery

This section covers how a Linux host determines where network traffic
should be sent.

Topics include:

- subnetting
- local vs remote destinations
- routing tables
- default gateways
- longest-prefix matching
- ARP / neighbor discovery
- Layer 2 vs Layer 3 addressing
- practical Linux routing commands
- connectivity troubleshooting

---

# 1. Subnets

A subnet determines which part of an IP address identifies the network
and which part identifies a host inside that network.

Example:

```text
192.168.1.20/24
```

IPv4 contains 32 bits.

```text
192      .168      .1        .20
8 bits    8 bits    8 bits    8 bits
```

A `/24` means the first 24 bits identify the network.

Therefore:

```text
Host IP:          192.168.1.20
Subnet:           192.168.1.0/24
Subnet mask:      255.255.255.0
```

Typical range:

```text
Network:          192.168.1.0
Usable hosts:     192.168.1.1 - 192.168.1.254
Broadcast:        192.168.1.255
```

---

# 2. CIDR and Host Bits

IPv4 contains 32 bits.

The number after `/` tells how many bits belong to the network.

```text
/24
```

means:

```text
24 network bits
8 host bits
```

Number of addresses:

```text
2^(host bits)
```

Example:

```text
/24

32 - 24 = 8 host bits

2^8 = 256 addresses
```

Traditionally:

```text
256 - network address - broadcast address
= 254 usable hosts
```

---

# 3. Common Subnet Sizes

| CIDR | Subnet Mask | Addresses | Typical Usable Hosts |
|---|---|---:|---:|
| `/24` | `255.255.255.0` | 256 | 254 |
| `/25` | `255.255.255.128` | 128 | 126 |
| `/26` | `255.255.255.192` | 64 | 62 |
| `/27` | `255.255.255.224` | 32 | 30 |
| `/28` | `255.255.255.240` | 16 | 14 |

---

# 4. Block Size

A quick way to determine subnet boundaries is:

```text
256 - interesting subnet-mask octet
```

Example:

```text
/26
Mask = 255.255.255.192

256 - 192 = 64
```

Therefore the subnet boundaries are:

```text
0
64
128
192
```

So:

```text
192.168.1.0/26
192.168.1.64/26
192.168.1.128/26
192.168.1.192/26
```

Example host:

```text
192.168.1.70/26
```

belongs to:

```text
Network:      192.168.1.64
Usable:       192.168.1.65 - 192.168.1.126
Broadcast:    192.168.1.127
```

---

# 5. Local vs Remote Destination

A host determines whether the destination belongs to one of its directly
connected networks.

Example:

```text
Host:
192.168.1.20/24
```

Destination:

```text
192.168.1.50
```

is inside:

```text
192.168.1.0/24
```

so it is directly reachable.

The destination:

```text
192.168.2.50
```

is outside that subnet and therefore requires routing.

Conceptually:

```text
Destination IP
      ↓
Does it belong to a directly connected subnet?
      │
      ├── YES → send directly
      │
      └── NO  → use another route/default gateway
```

---

# 6. Routing Table

Linux maintains a routing table describing how destinations should be
reached.

Display it using:

```bash
ip route
```

Example from this lab:

```text
default via 172.18.192.1 dev eth0 proto kernel

172.18.192.0/20 dev eth0
    proto kernel
    scope link
    src 172.18.192.230
```

This means:

```text
WSL IP:
172.18.192.230

Directly connected network:
172.18.192.0/20

Default gateway:
172.18.192.1

Interface:
eth0
```

---

# 7. Directly Connected Route

This route:

```text
172.18.192.0/20 dev eth0
```

means destinations inside this subnet can be sent directly through
`eth0`.

For example:

```text
172.18.195.10
```

belongs to the `/20` subnet and therefore does not require the default
gateway as its Layer-3 next hop.

---

# 8. Default Gateway

The default route in this lab is:

```text
default via 172.18.192.1 dev eth0
```

It means:

> If no more specific route matches the destination, send the packet
> to `172.18.192.1` using `eth0`.

Example:

```text
Destination: 8.8.8.8
```

does not belong to the directly connected WSL subnet.

Therefore:

```text
WSL
172.18.192.230
      ↓
Default gateway
172.18.192.1
      ↓
Remote networks
      ↓
8.8.8.8
```

---

# 9. Inspecting a Routing Decision

Use:

```bash
ip route get <destination>
```

Example:

```bash
ip route get 8.8.8.8
```

Output from this lab:

```text
8.8.8.8 via 172.18.192.1 dev eth0 src 172.18.192.230
```

Interpretation:

```text
Destination = 8.8.8.8
Next hop    = 172.18.192.1
Interface   = eth0
Source IP   = 172.18.192.230
```

This is useful when troubleshooting which network path Linux intends
to use.

---

# 10. Longest-Prefix Matching

Linux chooses the most specific route that matches the destination.

Example routing table:

```text
10.0.0.0/8      via gateway-A
10.10.0.0/16    via gateway-B
```

Destination:

```text
10.10.5.20
```

matches both routes.

Linux chooses:

```text
10.10.0.0/16
```

because `/16` is more specific than `/8`.

Therefore:

```text
gateway-B
```

is selected.

Important:

```text
Routing does not simply use the first matching entry.

It chooses the longest matching network prefix.
```

---

# 11. Layer 3 Destination vs Next Hop

When sending traffic to a remote destination such as:

```text
8.8.8.8
```

the Layer-3 destination remains:

```text
8.8.8.8
```

The default gateway is only the next hop.

Example:

```text
Destination IP = 8.8.8.8
Next-hop IP    = 172.18.192.1
```

The packet does not change its destination IP simply because it is being
sent through a router.

---

# 12. ARP / Neighbor Discovery

For IPv4, ARP is used to map a directly reachable IP address to a
Layer-2 MAC address.

Conceptually:

```text
IP known
    ↓
MAC unknown
    ↓
ARP request
    ↓
ARP reply
    ↓
MAC learned
    ↓
Ethernet frame can be sent
```

Example:

```text
Who has 192.168.1.50?
```

The destination may respond:

```text
192.168.1.50 is at aa:bb:cc:dd:ee:ff
```

---

# 13. ARP for Local Destinations

If the destination is on the same subnet:

```text
Host A
192.168.1.20
      ↓
Destination
192.168.1.50
```

Host A needs the destination host's MAC address.

Therefore:

```text
Layer 3 destination:
192.168.1.50

Layer 2 destination:
MAC address of 192.168.1.50
```

---

# 14. ARP for Remote Destinations

For a remote destination:

```text
8.8.8.8
```

the host does NOT ARP for `8.8.8.8`.

It ARPs for the next-hop gateway.

Example:

```text
Remote IP
8.8.8.8
      ↓

Layer 3 destination
8.8.8.8

Layer 2 destination
MAC of 172.18.192.1
```

The router receives the frame, examines the IP packet, selects its next
hop, and creates a new Layer-2 frame.

Layer-2 addresses can therefore change at every hop while the Layer-3
destination continues to identify the remote system.

---

# 15. Inspecting the Neighbor Table

Use:

```bash
ip neigh
```

Example from this lab:

```text
172.18.192.1 dev eth0 lladdr 00:15:5d:f6:a1:87 STALE
```

Interpretation:

```text
Neighbor IP:
172.18.192.1

Interface:
eth0

MAC:
00:15:5d:f6:a1:87

State:
STALE
```

---

# 16. Neighbor States

Common states include:

| State | Meaning |
|---|---|
| `REACHABLE` | Neighbor was recently confirmed |
| `STALE` | Mapping exists but was not recently confirmed |
| `DELAY` | Linux is waiting before probing |
| `PROBE` | Linux is actively checking reachability |
| `INCOMPLETE` | Address resolution is in progress |
| `FAILED` | Neighbor resolution failed |

`STALE` does not mean the route is broken.

The cached MAC address may still be used.

---

# 17. Ping Does Not Prove Overall Connectivity

During this lab:

```bash
ping -c 1 172.18.192.1
```

returned no ICMP Echo Reply.

However:

```bash
curl -I https://example.com
```

returned:

```text
HTTP/2 200
```

This demonstrates an important troubleshooting principle:

```text
ping failure
≠
network failure
```

`ping` tests ICMP echo communication.

A host or gateway may choose not to respond to ICMP while still forwarding
other traffic normally.

In this case, working HTTPS demonstrated that the actual outbound network
path was functioning.

---

# 18. Troubleshoot the Actual Protocol

Do not depend only on:

```bash
ping
```

Instead test the protocol the application actually requires.

Examples:

```bash
curl https://example.com
```

tests HTTPS/application connectivity.

```bash
nc -vz example.com 443
```

can test TCP connectivity to port 443.

```bash
dig example.com
```

can test DNS resolution.

---

# 19. Networking Layers Used by HTTPS

A simplified HTTPS request stack is:

```text
Application data
      ↓
HTTP                      Layer 7
      ↓
TLS                       commonly mapped to Layer 6
      ↓
TCP                       Layer 4
      ↓
IP                        Layer 3
      ↓
Ethernet / Wi-Fi          Layer 2
```

HTTPS can therefore be thought of as:

```text
HTTP
over TLS
over TCP
over IP
```

---

# 20. DNS and TCP Are Separate

DNS resolution happens at the application layer.

Example:

```text
example.com
     ↓
DNS lookup
     ↓
IP address
```

After resolution, the client can attempt a TCP connection to the target
IP and port.

Therefore successful DNS does not prove TCP connectivity.

Similarly, successful TCP does not prove TLS or HTTP is working.

---

# 21. Layered Connectivity Troubleshooting

A useful mental model is:

```text
DNS
 ↓
Routing
 ↓
TCP
 ↓
TLS
 ↓
HTTP
 ↓
Application
```

Each stage proves something different.

For example:

```text
DNS succeeds
TCP fails
```

means DNS itself is probably not the problem.

Likewise:

```text
TCP succeeds
TLS certificate validation fails
```

means Layer-4 connectivity exists and troubleshooting should move higher
in the stack.

---

# Commands Used

| Command | Purpose |
|---|---|
| `ip route` | Display Linux routing table |
| `ip route get <IP>` | Show how Linux will route a destination |
| `ip addr` | Display network interfaces and IP addresses |
| `ip neigh` | Display IPv4 neighbor/ARP information |
| `ping` | Test ICMP Echo communication |
| `curl` | Test HTTP/HTTPS communication |

---

# Key Lessons

- The subnet determines whether a destination is directly reachable.
- Different subnets generally require routing.
- Linux chooses routes using longest-prefix matching.
- The default gateway is used when no more specific route matches.
- The Layer-3 destination remains the remote IP even when traffic is
  sent through a gateway.
- ARP maps directly reachable IPv4 addresses to MAC addresses.
- For remote traffic, the host resolves the gateway's MAC rather than
  the remote server's MAC.
- Layer-2 addresses change hop by hop.
- A failed ping does not automatically mean the network is down.
- Connectivity should be tested at the protocol layer actually used by
  the application.
- DNS, TCP, TLS, and HTTP are separate stages of communication.