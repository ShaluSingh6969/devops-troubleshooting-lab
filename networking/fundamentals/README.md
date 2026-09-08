# Networking Fundamentals for DevOps

## Core Layers

| Layer | Main Responsibility | Examples |
|---|---|---|
| Layer 2 | Local network delivery using MAC addresses | Ethernet, switches, ARP |
| Layer 3 | IP addressing and routing between networks | IPv4, routers |
| Layer 4 | End-to-end transport using ports | TCP, UDP |
| Layer 7 | Application protocols | HTTP, DNS, SSH |

## Same Subnet vs Different Subnet

### Same subnet

Traffic is normally sent directly to the destination host.

```text
Host A
  ↓
ARP for Host B MAC
  ↓
Host B
```

### Different subnet

```text
Host A
  ↓
default gateway
  ↓
router
  ↓
remote network
```

## IP vs Port
IP address -> which host? -> Layer 3
Port -> Which service/process -> Layer 4

## Binding
127.0.0.1:8080 - means the service is reachable only locally

0.0.0.0:8080 means the service listesn on all local IPv4 interfaces.
-  But this doesn't automatically make the service publicly reachable. Routing, NAT, and firewall rules still determine the rechability.