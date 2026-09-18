# Kubernetes Networking Fundamentals

## Objective

Understand how Kubernetes Pod networking works in a multi-node cluster, including:

- Node IPs vs Pod IPs
- Pod CIDRs
- kind networking
- CNI-managed Pod connectivity
- Pod network namespaces
- veth pairs
- same-node vs cross-node Pod communication
- node routing for Pod CIDRs
- direct Pod-to-Pod reachability

---

# 1. Lab Environment

The Kubernetes lab runs inside WSL2 Ubuntu using Docker Engine and kind.

Architecture:

```text
Windows
   ↓
WSL2 Ubuntu
   ↓
Docker Engine
   ↓
kind Kubernetes cluster
   ↓
Kubernetes Nodes
   ↓
Pods
```

The kind cluster contains:

```text
devops-lab-control-plane
devops-lab-worker
devops-lab-worker2
```

The Kubernetes nodes are themselves Docker containers.

---

# 2. Kubernetes API Access

The cluster showed:

```text
Kubernetes control plane:
https://127.0.0.1:42873
```

Docker showed:

```text
127.0.0.1:42873 -> 6443/tcp
```

So kubectl reaches the Kubernetes API server using:

```text
WSL localhost:42873
        ↓
Docker port publishing
        ↓
control-plane:6443
```

---

# 3. Kubernetes Nodes as Docker Containers

Command:

```bash
docker ps
```

showed:

```text
devops-lab-control-plane
devops-lab-worker
devops-lab-worker2
```

The Kubernetes nodes use containerd internally:

```text
containerd://2.3.4
```

So there are two different container layers:

```text
Docker
→ runs the kind Kubernetes nodes

containerd inside each kind node
→ runs Kubernetes Pods
```

---

# 4. kind Docker Network

The kind nodes are connected to a Docker user-defined bridge network.

Command:

```bash
docker network inspect kind \
  --format '{{range .Containers}}{{.Name}} -> {{.IPv4Address}}{{println}}{{end}}'
```

Observed:

```text
devops-lab-control-plane -> 172.21.0.3/16
devops-lab-worker        -> 172.21.0.2/16
devops-lab-worker2       -> 172.21.0.4/16
```

WSL routing table included:

```text
172.21.0.0/16 dev br-92c88036c640 proto kernel scope link src 172.21.0.1
```

So:

```text
172.21.0.0/16
```

is the Docker network containing the Kubernetes nodes.

The bridge gateway is:

```text
172.21.0.1
```

---

# 5. WSL Routing Table

Observed:

```text
default via 172.18.192.1 dev eth0

172.17.0.0/16 dev docker0
172.18.192.0/20 dev eth0
172.19.0.0/16 dev br-60f51c52d4a7
172.20.0.0/16 dev br-9616c78e7540
172.21.0.0/16 dev br-92c88036c640
```

Important distinction:

```text
172.18.192.1
→ WSL default gateway

172.21.0.1
→ gateway of Docker's kind network
```

---

# 6. Node IPs and Pod CIDRs

Command:

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,NODE-IP:.status.addresses[?(@.type=="InternalIP")].address,POD-CIDR:.spec.podCIDR'
```

Observed:

```text
NODE                       NODE-IP      POD-CIDR
devops-lab-control-plane   172.21.0.3   10.244.0.0/24
devops-lab-worker          172.21.0.2   10.244.2.0/24
devops-lab-worker2         172.21.0.4   10.244.1.0/24
```

This creates two different network layers:

```text
Node network:
172.21.0.0/16

Pod networks:
10.244.0.0/24
10.244.1.0/24
10.244.2.0/24
```

---

# 7. Pod CIDRs per Node

Each node owns a Pod CIDR.

```text
control-plane
Node IP: 172.21.0.3
Pod CIDR: 10.244.0.0/24
```

```text
worker
Node IP: 172.21.0.2
Pod CIDR: 10.244.2.0/24
```

```text
worker2
Node IP: 172.21.0.4
Pod CIDR: 10.244.1.0/24
```

Pods scheduled to a node receive addresses from that node's Pod CIDR.

---

# 8. Kubernetes Networking Components

Command:

```bash
kubectl get daemonsets -n kube-system
```

showed:

```text
kindnet
kube-proxy
```

The cluster also contains CoreDNS Pods.

Conceptually:

```text
kindnet
→ Pod networking / Pod CIDR connectivity

kube-proxy
→ Service traffic handling

CoreDNS
→ cluster DNS
```

---

# 9. kindnet Pods

Command:

```bash
kubectl get pods -n kube-system -o wide
```

showed one kindnet Pod per node.

Example:

```text
kindnet-jckzj   → devops-lab-worker
kindnet-vc74r   → devops-lab-worker2
kindnet-pln99   → devops-lab-control-plane
```

This is because kindnet runs as a DaemonSet.

---

# 10. Node Routing Tables

Inside `devops-lab-worker`:

```bash
docker exec -it devops-lab-worker bash
ip route
```

Observed:

```text
default via 172.21.0.1 dev eth0
10.244.0.0/24 via 172.21.0.3 dev eth0
10.244.1.0/24 via 172.21.0.4 dev eth0
172.21.0.0/16 dev eth0 proto kernel scope link src 172.21.0.2
```

This means worker1 knows:

```text
Pods in 10.244.0.0/24
→ reachable through control-plane node 172.21.0.3

Pods in 10.244.1.0/24
→ reachable through worker2 node 172.21.0.4
```

Inside worker2:

```text
default via 172.21.0.1 dev eth0
10.244.0.0/24 via 172.21.0.3 dev eth0
10.244.2.0/24 via 172.21.0.2 dev eth0
172.21.0.0/16 dev eth0 proto kernel scope link src 172.21.0.4
```

So worker2 knows:

```text
Pods in 10.244.2.0/24
→ reachable through worker1 node 172.21.0.2
```

---

# 11. Cross-Node Routing

Suppose:

```text
Pod A
10.244.2.2
Node: devops-lab-worker
```

and:

```text
Pod B
10.244.1.2
Node: devops-lab-worker2
```

Worker1 has:

```text
10.244.1.0/24 via 172.21.0.4 dev eth0
```

So traffic to Pod B follows:

```text
Pod A
10.244.2.2
   ↓
worker1
   ↓
route to 10.244.1.0/24
   ↓
next hop 172.21.0.4
   ↓
worker2
   ↓
Pod B
10.244.1.2
```

---

# 12. Practical Cross-Node Pod Test

Two Pods were created:

```text
pod-a
IP: 10.244.2.2
Node: devops-lab-worker
```

```text
pod-b
IP: 10.244.1.2
Node: devops-lab-worker2
```

Command:

```bash
kubectl get pods -o wide
```

confirmed the node placement and IPs.

Then from Pod A:

```bash
kubectl exec -it pod-a -- ping -c 3 10.244.1.2
```

worked successfully.

Observed:

```text
3 packets transmitted
3 received
0% packet loss
```

This proved direct cross-node Pod-to-Pod communication.

---

# 13. Pod Network Namespace

Inside Pod A:

```bash
kubectl exec -it pod-a -- ip addr
```

showed:

```text
eth0
10.244.2.2/24
```

The Pod has its own network namespace with its own:

```text
interfaces
IP addresses
routing table
localhost
port space
```

---

# 14. Pod Routing Table

Inside Pod A:

```bash
kubectl exec -it pod-a -- ip route
```

showed:

```text
default via 10.244.2.1 dev eth0
10.244.2.0/24 via 10.244.2.1 dev eth0 src 10.244.2.2
10.244.2.1 dev eth0 scope link src 10.244.2.2
```

So Pod A does not directly contain a route for:

```text
10.244.1.0/24
```

Instead, it sends non-local traffic to:

```text
10.244.2.1
```

which is its gateway into the node networking stack.

---

# 15. Pod-to-Node veth Pair

On `devops-lab-worker`, the following interface was found:

```text
vetha80f48b3@if2
```

with:

```text
inet 10.244.2.1/32
```

and:

```text
link-netns cni-a2b3a81c-b495-0925-5976-408e79903b5c
```

This is the node-side interface connected to Pod A.

Conceptually:

```text
Pod A namespace

eth0
10.244.2.2
      ||
      || veth pair
      ||
Node namespace

vetha80f48b3
10.244.2.1
```

---

# 16. Meaning of @ifX

Example inside the kind worker:

```text
eth0@if25
```

means:

```text
local interface:
eth0

local interface index:
2

peer interface index:
25
```

Similarly:

```text
vetha80f48b3@if2
```

means the local node-side interface has a peer whose interface index is `2` in another network namespace.

The `@ifX` value is:

```text
peer interface index
```

not:

```text
IP address
port
subnet
```

---

# 17. Node-to-WSL veth Pair

Inside `devops-lab-worker`:

```text
2: eth0@if25
    inet 172.21.0.2/16
```

On the WSL host:

```text
25: vethaa4b5f0@if2
    master br-92c88036c640
```

These are the two ends of the same veth pair.

Conceptually:

```text
WSL namespace

vethaa4b5f0
index 25
      ||
      || veth pair
      ||
kind worker namespace

eth0
index 2
172.21.0.2
```

---

# 18. Docker kind Bridge

The WSL-side veth showed:

```text
master br-92c88036c640
```

This means it is attached to the Linux bridge backing Docker's kind network.

So:

```text
br-92c88036c640
```

connects the kind node containers together.

Conceptually:

```text
WSL host

br-92c88036c640
172.21.0.1
   │
   ├── control-plane 172.21.0.3
   ├── worker        172.21.0.2
   └── worker2       172.21.0.4
```

---

# 19. Full Nested Network Path

The full observed architecture is:

```text
WSL HOST

br-92c88036c640
172.21.0.1
   ↓
host-side veth
   ↓

KIND WORKER

eth0
172.21.0.2
   ↓
Linux routing
   ↓
vetha80f48b3
10.244.2.1
   ↓

POD A

eth0
10.244.2.2
```

So there are two different veth boundaries:

```text
WSL
↔
Kubernetes Node
```

and:

```text
Kubernetes Node
↔
Pod
```

---

# 20. Packet Path from Pod A to Pod B

Pod A:

```text
10.244.2.2
```

wants to reach Pod B:

```text
10.244.1.2
```

Pod A first checks its routing table.

Since `10.244.1.2` is not local, it chooses:

```text
default via 10.244.2.1
```

Packet travels:

```text
Pod A eth0
10.244.2.2
   ↓
veth pair
   ↓
vetha80f48b3
10.244.2.1
```

The packet is now inside worker1's Linux networking stack.

Worker1 checks:

```text
10.244.1.0/24 via 172.21.0.4 dev eth0
```

Since:

```text
10.244.1.2 ∈ 10.244.1.0/24
```

worker1 selects:

```text
next hop = 172.21.0.4
interface = eth0
```

The packet then travels:

```text
worker1
172.21.0.2
   ↓
kind Docker bridge
   ↓
worker2
172.21.0.4
```

Worker2 then delivers the packet into its local Pod network.

Final path:

```text
Pod A
10.244.2.2
   ↓
gateway 10.244.2.1
   ↓
worker1 routing
   ↓
172.21.0.4
   ↓
worker2
   ↓
Pod B
10.244.1.2
```

---

# 21. Why Pod-to-Pod Communication Works

Pods do not need a Service to communicate directly.

A Pod can reach another Pod's current IP directly if the cluster networking is working.

For example:

```text
10.244.2.2
→
10.244.1.2
```

worked successfully.

A Service is still normally used by applications because Pod IPs can change when Pods are recreated.

So:

```text
Pod IP
→ direct networking capability

Service
→ stable application endpoint
```

---

# 22. Same Cluster Requirement

The Pod-to-Pod model applies within the same Kubernetes cluster.

Nodes in the same cluster know how to reach each other's Pod CIDRs.

Example:

```text
worker1 knows:
10.244.1.0/24 via 172.21.0.4

worker2 knows:
10.244.2.0/24 via 172.21.0.2
```

Different Kubernetes clusters do not automatically know each other's Pod networks.

Cross-cluster communication needs additional networking.

---

# 23. Important ip addr Fields

Example:

```text
vetha80f48b3@if2:
<BROADCAST,MULTICAST,UP,LOWER_UP>
mtu 1500
qdisc noqueue
state UP
```

Meaning:

```text
vetha80f48b3
→ interface name

@if2
→ peer interface index

BROADCAST
→ supports Ethernet broadcast

MULTICAST
→ supports multicast

UP
→ administratively enabled

LOWER_UP
→ underlying link is operational

mtu 1500
→ maximum normal packet size

qdisc noqueue
→ no traditional transmit queue

state UP
→ operational state is up
```

---

# 24. Layer-2 Information

Example:

```text
link/ether 8a:88:55:fc:d2:37
```

means:

```text
Ethernet MAC address
```

and:

```text
brd ff:ff:ff:ff:ff:ff
```

is the Ethernet broadcast address.

---

# 25. IPv4 Address Information

Example:

```text
inet 10.244.2.1/32 scope global
```

means:

```text
inet
→ IPv4

10.244.2.1
→ assigned address

/32
→ single host route/address

scope global
→ usable beyond host/link-local scope
```

`global` does not mean public Internet IP.

---

# 26. IPv6 Link-Local Address

Example:

```text
inet6 fe80::.../64 scope link
```

means:

```text
IPv6 link-local address
```

`fe80::/10` addresses are valid only on the local link.

---

# 27. Troubleshooting Cross-Node Pod Connectivity

If Pod A can reach a same-node Pod but cannot reach a Pod on another node, investigate:

```text
1. Does the destination Pod have a valid IP?

2. Is the destination Pod running?

3. Can the source Node reach the destination Node?

4. Does the source Node have the correct route for the destination Pod CIDR?

5. Is the CNI component healthy?

6. Is NetworkPolicy blocking traffic?

7. Is the destination process actually listening?
```

Useful commands:

```bash
kubectl get pods -o wide
```

```bash
kubectl get nodes -o wide
```

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,NODE-IP:.status.addresses[?(@.type=="InternalIP")].address,POD-CIDR:.spec.podCIDR'
```

```bash
docker exec -it <kind-node> bash
```

```bash
ip addr
```

```bash
ip route
```

```bash
ip route get <pod-ip>
```

```bash
kubectl exec -it <pod> -- ip addr
```

```bash
kubectl exec -it <pod> -- ip route
```

```bash
kubectl exec -it <pod> -- ping <destination-pod-ip>
```

---

# 28. Final Mental Model

The main packet path is:

```text
Pod
   ↓
Pod eth0
   ↓
veth pair
   ↓
Node-side veth
   ↓
Node routing table
   ↓
destination Node
   ↓
destination Node routing
   ↓
destination Pod veth
   ↓
destination Pod
```

The important routing responsibility is split between:

```text
Pod routing table
→ sends non-local traffic to Pod gateway

Node routing table
→ knows how to reach other Pod CIDRs
```

---

# Key Takeaways

- kind Kubernetes nodes are Docker containers.
- Kubernetes Pods run inside those kind nodes.
- Node IPs and Pod IPs belong to different networks.
- Each node owns a Pod CIDR.
- Pod traffic enters the node through a veth pair.
- Node routing decides how to reach remote Pod CIDRs.
- Cross-node Pod traffic can be inspected directly with Linux routing tools.
- Pod-to-Pod communication does not require a Service.
- Services are mainly used to provide stable application endpoints.
- `ip route get <pod-ip>` is extremely useful for debugging Pod routing.
- veth pairs appear at both the WSL-to-kind-node boundary and the node-to-Pod boundary.