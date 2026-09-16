# Docker Networking Fundamentals in WSL

## Objective

Understand how Docker networking works inside WSL, how containers reach external networks, and how Docker-created subnets participate in the Linux routing table.

The main topics covered were:

- WSL networking vs Windows networking
- Docker bridge networks
- `docker0`
- container network namespaces
- `veth` pairs
- container routing tables
- WSL host routing
- Docker subnet overlap
- how Linux selects the next hop

---

# Overall Architecture

In the current setup, networking is layered:

```text
Windows host
   ↓
WSL2 Linux environment
   ↓
Docker networking
   ↓
Containers
```

The Windows host owns the physical Wi-Fi/Ethernet connection.

WSL has its own virtual network interface and routing table.

Docker then creates additional virtual networks inside the Linux environment.

---

# Actual WSL Network

The WSL routing table showed:

```text
default via 172.18.192.1 dev eth0

172.17.0.0/16 dev docker0
172.18.192.0/20 dev eth0
172.19.0.0/16 dev br-60f51c52d4a7
```

The WSL interface is:

```text
eth0 = 172.18.192.230
```

The WSL default gateway is:

```text
172.18.192.1
```

So normal WSL traffic to an external destination follows approximately:

```text
WSL application
   ↓
eth0
172.18.192.230
   ↓
default gateway
172.18.192.1
   ↓
Windows-managed networking
   ↓
physical network / VPN / Internet
```

---

# Docker Networks

Docker showed the following networks:

```text
bridge
host
none
scc_compute_task_default
```

The default bridge network was:

```text
Subnet:  172.17.0.0/16
Gateway: 172.17.0.1
```

The corresponding Linux bridge interface is:

```text
docker0
```

with:

```text
docker0 = 172.17.0.1/16
```

A second Docker bridge network existed:

```text
scc_compute_task_default
```

with corresponding Linux bridge:

```text
br-60f51c52d4a7
```

and route:

```text
172.19.0.0/16 dev br-60f51c52d4a7
```

---

# What `docker0` Is

`docker0` is a Linux bridge created by Docker.

It acts roughly like a virtual Layer-2 switch for containers connected to the default bridge network.

Conceptually:

```text
                 WSL Host

              docker0
            172.17.0.1
              /      \
           veth      veth
            |          |
       container A  container B
```

It is not a physical interface.

---

# Container Network Namespace

Each container normally has its own network namespace.

This means the container has its own:

```text
interfaces
IP addresses
routing table
ports
```

The test container was:

```text
net-lab
```

and had:

```text
IP      = 172.17.0.2
Gateway = 172.17.0.1
```

Its routing table was:

```text
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 scope link src 172.17.0.2
```

So the container sees:

```text
local subnet:
172.17.0.0/16

default gateway:
172.17.0.1
```

---

# veth Pair

Docker connects the container network namespace to the host using a virtual Ethernet pair.

The host showed:

```text
veth24074fe@if2
```

The container showed:

```text
eth0@if5
```

These are the two ends of the same virtual Ethernet connection.

Conceptually:

```text
Container namespace                WSL host namespace

eth0
172.17.0.2
   |
   | virtual Ethernet pair
   |
veth24074fe
   |
   |
docker0
172.17.0.1
```

A useful mental model is:

```text
veth pair = virtual Ethernet cable

docker0 = virtual switch/bridge
```

---

# Why `docker0` Changed State

Before a container was attached, `docker0` showed:

```text
NO-CARRIER
state DOWN
```

After starting the container, it showed:

```text
UP
LOWER_UP
state UP
```

This happened because an active veth interface was now attached to the bridge.

So `docker0` can exist with an IP address even when no container is connected.

---

# Container Routing Decision

Suppose the container wants to reach:

```text
8.8.8.8
```

Its routing table is:

```text
172.17.0.0/16 dev eth0
default via 172.17.0.1
```

The container checks:

```text
Does 8.8.8.8 belong to 172.17.0.0/16?
```

No.

Therefore it chooses:

```text
default via 172.17.0.1
```

So the first routing decision is:

```text
Container
172.17.0.2
   ↓
default gateway
172.17.0.1
```

The packet still contains:

```text
Source:      172.17.0.2
Destination: 8.8.8.8
```

The destination does not become `172.17.0.1`.

`172.17.0.1` is simply the next hop.

---

# When the Packet Reaches WSL

The Docker gateway:

```text
172.17.0.1
```

belongs to the WSL/Linux host on the `docker0` interface.

So when the container forwards the packet to its gateway, the packet enters the WSL host's networking stack.

The flow is:

```text
container eth0
   ↓
veth pair
   ↓
docker0
   ↓
WSL/Linux networking stack
```

At this point the WSL kernel performs another routing-table lookup using the final destination:

```text
8.8.8.8
```

---

# WSL Routing Decision

The WSL route table contains:

```text
default via 172.18.192.1 dev eth0

172.17.0.0/16 dev docker0
172.18.192.0/20 dev eth0
172.19.0.0/16 dev br-60f51c52d4a7
```

For destination:

```text
8.8.8.8
```

Linux checks:

```text
8.8.8.8 ∈ 172.17.0.0/16 ? no
8.8.8.8 ∈ 172.18.192.0/20 ? no
8.8.8.8 ∈ 172.19.0.0/16 ? no
```

The only matching route is:

```text
default via 172.18.192.1 dev eth0
```

Therefore the WSL host forwards the packet toward:

```text
172.18.192.1
```

---

# Complete Packet Path

For a container connecting to an external destination such as `8.8.8.8`:

```text
Container
172.17.0.2
   ↓
container routing table
   ↓
default via 172.17.0.1
   ↓
container eth0
   ↓
veth pair
   ↓
docker0
172.17.0.1
   ↓
WSL host routing table
   ↓
default via 172.18.192.1
   ↓
WSL eth0
172.18.192.230
   ↓
Windows-managed networking
   ↓
Windows routing / VPN / physical NIC
   ↓
external network
   ↓
destination
```

This means several independent routing decisions can occur:

```text
Container routing
        ↓
WSL/Linux routing
        ↓
Windows networking
        ↓
upstream routers
```

Each router only chooses the next hop.

It does not calculate the complete end-to-end route.

---

# Understanding a Route Entry

Example:

```text
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
```

This means:

```text
172.17.0.0/16
→ destination network covered by this route

dev docker0
→ send matching traffic through docker0

proto kernel
→ route was automatically created by the kernel

scope link
→ destination is directly reachable on this link;
  no separate gateway is required

src 172.17.0.1
→ preferred source IP when the host itself creates
  traffic using this route
```

The `src` value is not used to determine whether the route matches.

Route matching is based primarily on the destination prefix.

---

# Destination Matching vs Preferred Source

For route selection:

```text
destination prefix
→ determines whether a route matches

via / dev
→ determines next hop and output interface

src
→ preferred source IP for locally generated traffic
```

For example, for:

```text
Destination: 8.8.8.8
```

Linux checks:

```text
172.17.0.0/16       no
172.18.192.0/20     no
172.19.0.0/16       no
default /0          yes
```

Therefore:

```text
via 172.18.192.1 dev eth0
```

The presence of:

```text
src 172.17.0.1
```

on the Docker route does not affect that matching process.

---

# Longest Prefix Match

Linux prefers the most specific matching route.

Example:

```text
172.19.0.0/16 dev br-docker
default via gateway
```

For destination:

```text
172.19.20.50
```

both technically match:

```text
172.19.0.0/16
0.0.0.0/0
```

but `/16` is more specific than `/0`.

Therefore Linux selects:

```text
172.19.0.0/16 dev br-docker
```

---

# Docker Subnet Overlap

Docker networks participate in the normal Linux routing table.

This can cause problems if a Docker subnet overlaps with a corporate/VPN subnet.

Example:

```text
Corporate network:
172.19.0.0/16

Internal server:
172.19.20.50

Docker network:
172.19.0.0/16
```

The host routing table may contain:

```text
172.19.0.0/16 dev br-docker
```

When trying to reach:

```text
172.19.20.50
```

Linux chooses:

```text
Docker bridge
```

instead of the intended:

```text
VPN / corporate route
```

because the Docker route is a direct `/16` match.

The resulting path becomes:

```text
internal service IP
   ↓
wrong Docker bridge
   ↓
neighbor discovery / routing fails
   ↓
service unreachable
```

This can appear as errors such as:

```text
No route to host
timeout
unreachable
```

depending on the exact network behavior.

---

# Why Docker Subnet Conflicts Matter

A Docker network can therefore unintentionally hijack traffic intended for another network.

Useful commands:

```bash
ip route
```

```bash
ip route get <destination-ip>
```

```bash
docker network ls
```

```bash
docker network inspect <network>
```

```bash
ip link show type bridge
```

These help identify:

```text
which Docker networks exist
which subnets they use
which Linux bridges were created
which route Linux chooses
```

---

# Important Commands Used

## List Docker Networks

```bash
docker network ls
```

## Inspect Docker Default Bridge

```bash
docker network inspect bridge
```

## View Host Routing Table

```bash
ip route
```

## Show Linux Bridges

```bash
ip link show type bridge
```

## Inspect `docker0`

```bash
ip addr show docker0
```

## Start Test Container

```bash
docker run -d --name net-lab nginx:alpine
```

## Inspect Container IP and Gateway

```bash
docker inspect net-lab \
  --format '{{range .NetworkSettings.Networks}}IP={{.IPAddress}} Gateway={{.Gateway}} NetworkID={{.NetworkID}}{{end}}'
```

Observed:

```text
IP=172.17.0.2
Gateway=172.17.0.1
```

## View Host Interfaces

```bash
ip link
```

## View Container Interfaces

```bash
docker exec net-lab ip addr
```

## View Container Routing Table

```bash
docker exec net-lab ip route
```

Observed:

```text
default via 172.17.0.1 dev eth0

172.17.0.0/16 dev eth0 scope link src 172.17.0.2
```

---

# Key Mental Model

The final networking model from this session is:

```text
CONTAINER NETWORK NAMESPACE

Container
172.17.0.2
   ↓
container route lookup
   ↓
gateway 172.17.0.1
   ↓


WSL / LINUX HOST

docker0
172.17.0.1
   ↓
WSL route lookup
   ↓
eth0
172.18.192.230
   ↓
gateway 172.18.192.1
   ↓


WINDOWS NETWORKING

Windows virtual networking / NAT
   ↓
Windows route selection
   ↓
Wi-Fi / Ethernet / VPN
   ↓


EXTERNAL NETWORK

router / corporate network / Internet
   ↓
destination
```

The key lesson is:

> Every networking layer makes its own routing decision based on the final destination IP and its own routing table.

Docker does not bypass normal Linux routing.

Docker-created bridge networks become real Linux routes and can affect how the host reaches other networks.

---

# Next Topic

The next step is to understand:

```text
Docker NAT / masquerading
```

because an external network normally has no route back to:

```text
172.17.0.2
```

We therefore need to understand how Docker rewrites the container's source address when traffic leaves the Docker network, and how replies are mapped back to the correct container.

After that:

```text
Docker port publishing
-p hostPort:containerPort
```

will explain how traffic travels in the opposite direction:

```text
host / external client
        ↓
published port
        ↓
Docker NAT
        ↓
container
```

# Docker Networking — NAT, Port Publishing, DNS, and Troubleshooting

## Objective

This section covers how Docker containers communicate with external networks, with the host, and with other containers.

Topics covered:

- Docker masquerading / SNAT
- why containers need NAT for Internet access
- connection tracking
- reverse NAT
- Docker port publishing
- `EXPOSE` vs `-p`
- Docker internal DNS
- external DNS forwarding
- container-to-container communication
- Docker network isolation
- controlled networking failures
- Docker subnet overlap

---

# 1. Container Egress Recap

Current default Docker network:

```text
Docker subnet:   172.17.0.0/16
Docker gateway:  172.17.0.1
Container IP:    172.17.0.2
```

WSL network:

```text
WSL IP:          172.18.192.230
WSL gateway:     172.18.192.1
```

Container routing:

```text
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0
```

WSL routing includes:

```text
default via 172.18.192.1 dev eth0
172.17.0.0/16 dev docker0
```

For a destination such as:

```text
8.8.8.8
```

the path is:

```text
Container
172.17.0.2
   ↓
default via 172.17.0.1
   ↓
docker0
   ↓
WSL routing table
   ↓
default via 172.18.192.1
   ↓
external network
```

---

# 2. Why NAT Is Required

A container may send:

```text
src = 172.17.0.2
dst = 8.8.8.8
```

But external networks normally do not have a route back to:

```text
172.17.0.0/16
```

because this is an internal Docker network.

Therefore Docker normally performs source NAT.

---

# 3. Docker Masquerading

Docker's default bridge showed:

```text
com.docker.network.bridge.enable_ip_masquerade = true
```

Masquerading is a form of source NAT.

Conceptually:

```text
BEFORE NAT

src = 172.17.0.2
dst = external-server
```

becomes:

```text
AFTER NAT

src = 172.18.192.230
dst = external-server
```

The exact address visible further outside may change again through WSL, Windows, or another router.

The important Docker-level principle is:

```text
container source IP
        ↓
masquerade
        ↓
host-side routable source IP
```

---

# 4. SNAT vs MASQUERADE

General source NAT:

```text
SNAT
→ explicitly rewrite source to a configured address
```

Masquerading:

```text
MASQUERADE
→ use the outgoing interface's address dynamically
```

Docker commonly uses masquerading for container egress.

---

# 5. Controlled Test — Masquerading Disabled

A custom network was created:

```bash
docker network create \
  --driver bridge \
  --opt com.docker.network.bridge.enable_ip_masquerade=false \
  no-masq-net
```

A container was started:

```bash
docker run -d \
  --name no-masq-test \
  --network no-masq-net \
  alpine sleep 1d
```

Observed:

```text
IP=172.20.0.2
Gateway=172.20.0.1
```

Container routes:

```text
default via 172.20.0.1 dev eth0
172.20.0.0/16 dev eth0 scope link src 172.20.0.2
```

Gateway test:

```bash
docker exec no-masq-test ping -c 3 172.20.0.1
```

worked.

External test:

```bash
docker exec no-masq-test ping -c 3 8.8.8.8
```

returned:

```text
100% packet loss
```

This demonstrated:

```text
container → Docker gateway
✓

external return traffic
✗
```

The problem was not local routing.

The problem was that the container source address was not being masqueraded for external communication.

---

# 6. Connection Tracking

Linux keeps state about active network flows using conntrack.

For example:

```text
172.17.0.2:59434
        →
151.101.6.132:80
```

after NAT may appear externally as:

```text
172.18.192.230:59434
        →
151.101.6.132:80
```

Linux remembers this relationship.

When the reply returns:

```text
151.101.6.132:80
        →
172.18.192.230:59434
```

conntrack knows the original destination should be:

```text
172.17.0.2:59434
```

Then reverse NAT is applied.

---

# 7. Observed Conntrack Entry

Command:

```bash
sudo conntrack -L | grep 172.17.0.2
```

Example observed:

```text
tcp 6 77 TIME_WAIT
src=172.17.0.2 dst=151.101.6.132 sport=59434 dport=80
src=151.101.6.132 dst=172.18.192.230 sport=80 dport=59434
[ASSURED]
```

Original direction:

```text
172.17.0.2:59434
        →
151.101.6.132:80
```

Reply direction visible after NAT:

```text
151.101.6.132:80
        →
172.18.192.230:59434
```

This is direct evidence that masquerading occurred.

---

# 8. Conntrack Fields

Example:

```text
tcp 6 77 TIME_WAIT ...
```

means:

```text
tcp
→ transport protocol

6
→ IP protocol number for TCP

77
→ remaining lifetime of conntrack entry

TIME_WAIT
→ connection has closed but state remains temporarily
```

`[ASSURED]` indicates conntrack has seen valid bidirectional traffic and considers the flow established/confirmed.

---

# 9. Multiple Containers Sharing One Host IP

Two containers can use the same host-side IP because NAT can also translate ports.

Example:

```text
Container A
172.17.0.2:50000
→
172.18.192.230:50000

Container B
172.17.0.3:50000
→
172.18.192.230:50001
```

Conntrack remembers both mappings.

Therefore return traffic can be delivered to the correct container.

---

# 10. Routing vs NAT vs Conntrack

These are different mechanisms.

```text
Routing
→ decides where the packet goes

NAT
→ changes IP addresses and/or ports

Conntrack
→ remembers active flows and NAT state
```

A valid route does not replace NAT.

A valid NAT rule does not replace routing.

---

# 11. Docker Port Publishing

Example:

```bash
docker run -d \
  --name web-test \
  -p 8080:80 \
  nginx:alpine
```

Syntax:

```text
-p HOST_PORT:CONTAINER_PORT
```

Therefore:

```text
8080
→ host-side published port

80
→ application port inside the container
```

Conceptually:

```text
client
   ↓
host:8080
   ↓
Docker forwarding
   ↓
container:80
   ↓
nginx
```

---

# 12. Published Port vs Container Port

If the container is:

```text
172.17.0.3
```

then both of these can be valid from the WSL host:

```text
localhost:8080
```

and:

```text
172.17.0.3:80
```

But:

```text
172.17.0.3:8080
```

normally fails because the container itself is listening on port `80`, not `8080`.

---

# 13. Inspect Port Publishing

Configured port mapping:

```bash
docker port web-test
```

or:

```bash
docker inspect web-test \
  --format '{{json .NetworkSettings.Ports}}'
```

Example:

```text
80/tcp → 0.0.0.0:8080
```

`docker port` shows configuration.

`conntrack` shows active flows.

These answer different questions.

---

# 14. Conntrack for Published-Port Traffic

Observed:

```text
tcp 6 101 TIME_WAIT
src=172.17.0.1 dst=172.17.0.3 sport=52428 dport=80
src=172.17.0.3 dst=172.17.0.1 sport=80 dport=52428
[ASSURED]
```

This showed the container-side leg:

```text
172.17.0.1:52428
        →
172.17.0.3:80
```

and the return:

```text
172.17.0.3:80
        →
172.17.0.1:52428
```

The host-side port `8080` was not visible in the Ubuntu WSL conntrack table.

In Docker Desktop + WSL2, part of the published-port path may be handled outside the Ubuntu WSL namespace.

Therefore:

```text
docker port
→ best for configured mapping

conntrack
→ best for active flow state
```

---

# 15. Inspect Traffic on docker0

Command:

```bash
sudo tcpdump -i docker0 -nn tcp port 80
```

Then:

```bash
curl http://localhost:8080
```

On `docker0`, traffic should be visible toward:

```text
container-ip:80
```

not necessarily host port:

```text
8080
```

because port publishing has already been handled before the traffic reaches the container-facing side.

---

# 16. EXPOSE vs -p

Dockerfile:

```dockerfile
EXPOSE 80
```

does not publish the port.

It is metadata describing the intended container port.

Actual publishing requires:

```bash
docker run -p 8080:80 ...
```

Summary:

```text
EXPOSE
→ image metadata/documentation

-p
→ explicit runtime port publishing

-P
→ automatically publish exposed ports to host ports
```

---

# 17. User-Defined Docker Networks

A custom network was created:

```bash
docker network create app-net
```

Containers:

```bash
docker run -d \
  --name web-a \
  --network app-net \
  nginx:alpine
```

```bash
docker run -d \
  --name client-a \
  --network app-net \
  alpine sleep 1d
```

Containers on the same user-defined Docker network can communicate directly.

---

# 18. Docker Internal DNS

Inside `client-a`:

```bash
docker exec client-a cat /etc/resolv.conf
```

showed:

```text
nameserver 127.0.0.11
options ndots:0

ExtServers: [host(10.255.255.254)]
```

`127.0.0.11` is the DNS resolver the container queries.

For a Docker-internal name:

```text
client-a
   ↓
DNS query: web-a
   ↓
127.0.0.11
   ↓
Docker resolves internal record
   ↓
172.20.0.2
```

Observed:

```bash
docker exec client-a getent hosts web-a
```

returned:

```text
172.20.0.2 web-a web-a
```

---

# 19. External DNS Forwarding

For an Internet name such as:

```text
google.com
```

Docker's DNS resolver can forward the request upstream.

In this environment:

```text
127.0.0.11
   ↓
10.255.255.254
   ↓
upstream DNS infrastructure
```

The container itself still queries only:

```text
127.0.0.11
```

Docker handles forwarding.

---

# 20. DNS Failure When Container Leaves Network

`web-a` was disconnected:

```bash
docker network disconnect app-net web-a
```

Then:

```bash
docker exec client-a dig web-a
```

returned:

```text
status: NXDOMAIN
ANSWER: 0
SERVER: 127.0.0.11#53
```

This demonstrated that Docker could no longer resolve `web-a` internally.

The unresolved query was then forwarded toward the host DNS path.

---

# 21. Proving DNS Forwarding with tcpdump

Command:

```bash
sudo tcpdump -i any -nn \
  host 10.255.255.254 and port 53
```

Then:

```bash
docker exec client-a dig web-a
```

Observed:

```text
10.255.255.254.48097 > 10.255.255.254.53:
A? web-a.

10.255.255.254.53 > 10.255.255.254.48097:
NXDomain
```

This proved that the unresolved Docker name reached the host DNS path.

The final logic was:

```text
client-a
   ↓
127.0.0.11
   ↓
Docker cannot resolve web-a internally
   ↓
10.255.255.254
   ↓
host/upstream DNS
   ↓
NXDOMAIN
```

---

# 22. Docker DNS Troubleshooting

If a container name fails:

```bash
docker exec <container> cat /etc/resolv.conf
```

Check:

```text
nameserver 127.0.0.11
```

Then:

```bash
docker exec <container> getent hosts <service-name>
```

or:

```bash
docker exec <container> dig <service-name>
```

Then verify network membership:

```bash
docker network inspect <network>
```

Questions to ask:

```text
Are both containers on the same network?

Does Docker DNS resolve the service?

Does external DNS still work?

Was the service disconnected?

Is the service listening on the expected port?
```

---

# 23. Wrong-Port Failure

If:

```text
web-a → 172.20.0.2
```

resolves correctly but:

```bash
wget http://web-a:8080
```

fails while:

```bash
wget http://web-a:80
```

works, then DNS is not the problem.

Troubleshooting result:

```text
DNS
✓

network membership
✓

TCP port
✗
```

---

# 24. Different Docker Networks

Containers on different user-defined networks do not automatically share Docker DNS visibility.

Example:

```text
web-a
   |
app-net


isolated-client
   |
isolated-net
```

Then:

```text
isolated-client → web-a
```

may fail to resolve.

But:

```text
isolated-client → example.com
```

may still resolve through upstream DNS.

This distinguishes:

```text
Docker internal DNS problem
```

from:

```text
complete DNS failure
```

---

# 25. Docker Subnet Overlap

A simulated corporate destination was:

```text
172.30.50.10
```

Before creating the conflicting Docker network:

```bash
ip route get 172.30.50.10
```

returned:

```text
172.30.50.10 via 172.18.192.1 dev eth0
src 172.18.192.230
```

Then a conflicting Docker network was created:

```bash
docker network create \
  --driver bridge \
  --subnet 172.30.0.0/16 \
  overlap-net
```

Afterward:

```bash
ip route get 172.30.50.10
```

returned:

```text
172.30.50.10 dev br-e05e3a60235d
src 172.30.0.1
```

Docker had created:

```text
172.30.0.0/16 dev br-e05e3a60235d
```

Since:

```text
172.30.50.10 ∈ 172.30.0.0/16
```

Linux selected the Docker route instead of the default route.

---

# 26. Why the Docker Route Wins

Linux uses longest-prefix matching.

For:

```text
172.30.50.10
```

both routes match:

```text
172.30.0.0/16
0.0.0.0/0
```

But:

```text
/16
```

is more specific than:

```text
/0
```

Therefore:

```text
Docker bridge wins
```

This can cause traffic intended for a VPN or corporate network to be sent into the wrong Docker bridge.

---

# 27. Diagnose Subnet Overlap

The fastest command is:

```bash
ip route get <destination-ip>
```

Example:

```bash
ip route get 172.30.50.10
```

If the expected route is:

```text
VPN / eth0 / corporate gateway
```

but the result shows:

```text
dev br-xxxxxxxx
```

then Docker networking may be hijacking the route.

Then inspect:

```bash
docker network ls
```

and:

```bash
docker network inspect <network>
```

Look for overlapping subnets.

---

# 28. Fix Docker Subnet Overlap

Remove the conflicting network:

```bash
docker network rm overlap-net
```

Then recreate it using a non-conflicting subnet:

```bash
docker network create \
  --driver bridge \
  --subnet 172.25.0.0/16 \
  overlap-net
```

The exact replacement subnet must be chosen so that it does not overlap with:

```text
corporate networks
VPN networks
WSL networks
local LAN
other Docker networks
cloud/private networks
```

After fixing:

```bash
ip route get 172.30.50.10
```

should again point toward the intended external/VPN route.

---

# 29. Docker Compose Subnet Configuration

Example:

```yaml
services:
  app:
    image: my-app
    networks:
      - app-net

networks:
  app-net:
    ipam:
      config:
        - subnet: 172.25.0.0/16
```

The Docker network normally needs to be recreated after changing its subnet.

Example:

```bash
docker compose down
docker compose up -d
```

---

# 30. Docker Default Address Pools

If Docker frequently allocates conflicting subnets, configure Docker's default address pools.

Example concept:

```json
{
  "default-address-pools": [
    {
      "base": "172.25.0.0/16",
      "size": 24
    }
  ]
}
```

This can make new Docker networks use ranges such as:

```text
172.25.0.0/24
172.25.1.0/24
172.25.2.0/24
```

instead of allocating potentially conflicting ranges.

In Docker Desktop, daemon configuration should be managed through Docker Desktop's supported configuration mechanism.

---

# 31. Useful Commands

```bash
docker network ls
```

List Docker networks.

```bash
docker network inspect <network>
```

Inspect subnet, gateway, connected containers, and options.

```bash
docker port <container>
```

Show configured published ports.

```bash
docker inspect <container>
```

Inspect container networking configuration.

```bash
ip route
```

Show Linux routing table.

```bash
ip route get <destination>
```

Show the actual route Linux would choose.

```bash
sudo conntrack -L
```

Show active tracked flows.

```bash
sudo tcpdump -i docker0 -nn
```

Observe traffic on the Docker bridge.

```bash
docker exec <container> cat /etc/resolv.conf
```

Inspect container DNS configuration.

```bash
docker exec <container> getent hosts <name>
```

Test application-style name resolution.

```bash
docker exec <container> dig <name>
```

Inspect DNS resolution directly.

---

# 32. Final Mental Model

Docker networking combines multiple Linux networking mechanisms:

```text
Container network namespace
        ↓
container routing
        ↓
veth pair
        ↓
Docker bridge
        ↓
host routing
        ↓
NAT / masquerading
        ↓
external network
```

For return traffic:

```text
external network
        ↓
host
        ↓
conntrack
        ↓
reverse NAT
        ↓
Docker bridge
        ↓
veth
        ↓
container
```

For Docker internal names:

```text
container
   ↓
127.0.0.11
   ↓
Docker internal DNS
```

For external names:

```text
container
   ↓
127.0.0.11
   ↓
host DNS path
10.255.255.254
   ↓
upstream DNS
```

For published ports:

```text
host:8080
   ↓
Docker forwarding
   ↓
container:80
```

For subnet overlap:

```text
destination IP
   ↓
Linux longest-prefix match
   ↓
wrong Docker bridge
   ↓
traffic hijacked
```

---

# Key Takeaways

- Docker containers use normal Linux routing.
- Docker bridge networks create real routes on the host.
- Masquerading allows private container addresses to communicate with external networks.
- Conntrack remembers NAT mappings and active flows.
- Reverse NAT depends on conntrack state.
- `-p HOST:CONTAINER` publishes a port.
- `EXPOSE` alone does not publish anything.
- User-defined Docker networks provide embedded DNS through `127.0.0.11`.
- Docker forwards unresolved external names toward upstream DNS.
- Containers on different networks may not resolve each other's names.
- `ip route get` is one of the fastest ways to diagnose subnet overlap.
- Docker subnet conflicts should normally be fixed by changing Docker network allocation rather than adding fragile host-specific routes.