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

# Kubernetes Networking & Service Troubleshooting Lab

## Objective

This lab documents the Kubernetes networking behavior observed in a local multi-node `kind` cluster running on Docker Engine inside WSL2.

Topics covered:

- kind node networking
- Pod CIDRs
- Pod network namespaces
- veth pairs
- cross-node Pod routing
- ClusterIP Services
- EndpointSlices
- kube-proxy
- iptables Service rules
- DNAT
- Service selector failures
- wrong `targetPort` failures
- CoreDNS
- Kubernetes DNS resolution
- NodePort Services
- `externalTrafficPolicy: Cluster`
- `externalTrafficPolicy: Local`

The main goal is not to memorize every Kubernetes chain name, but to understand:

> At which layer did the packet stop?

---

# 1. Lab Architecture

The environment is:

```text
Windows 11
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

The cluster contains:

```text
devops-lab-control-plane
devops-lab-worker
devops-lab-worker2
```

The kind nodes are Docker containers.

Inside those nodes, Kubernetes uses `containerd` to run Pods.

---

# 2. Node Network

The nodes use Docker's `kind` network.

Observed node IPs:

```text
devops-lab-control-plane   172.21.0.3
devops-lab-worker          172.21.0.2
devops-lab-worker2         172.21.0.4
```

The WSL host had:

```text
172.21.0.0/16 dev br-92c88036c640
```

with:

```text
172.21.0.1
```

as the bridge-side address.

So:

```text
172.21.0.0/16
```

is the node network.

---

# 3. Pod CIDRs

Each Kubernetes node owns a Pod CIDR.

Observed:

```text
control-plane
Node IP: 172.21.0.3
Pod CIDR: 10.244.0.0/24
```

```text
worker1
Node IP: 172.21.0.2
Pod CIDR: 10.244.2.0/24
```

```text
worker2
Node IP: 172.21.0.4
Pod CIDR: 10.244.1.0/24
```

Command:

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,NODE-IP:.status.addresses[?(@.type=="InternalIP")].address,POD-CIDR:.spec.podCIDR'
```

---

# 4. Cross-Node Pod Test

Two Pods were used:

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

Test:

```bash
kubectl exec -it pod-a -- ping -c 3 10.244.1.2
```

Result:

```text
0% packet loss
```

This proved direct Pod-to-Pod communication across nodes.

---

# 5. Pod Routing

Inside Pod A:

```bash
kubectl exec -it pod-a -- ip route
```

Observed:

```text
default via 10.244.2.1 dev eth0
10.244.2.0/24 via 10.244.2.1 dev eth0 src 10.244.2.2
10.244.2.1 dev eth0 scope link src 10.244.2.2
```

So Pod A sends non-local traffic to:

```text
10.244.2.1
```

which is the node-side gateway for the Pod.

---

# 6. Pod veth Pair

On worker1, the node-side interface connected to Pod A was:

```text
vetha80f48b3@if2
inet 10.244.2.1/32
```

Conceptually:

```text
Pod namespace

eth0
10.244.2.2
   ||
   || veth pair
   ||
Node namespace

vetha80f48b3
10.244.2.1
```

A veth pair acts like a virtual Ethernet cable connecting two network namespaces.

---

# 7. Node-to-WSL veth Pair

Inside worker1:

```text
eth0@if25
172.21.0.2/16
```

On the WSL host:

```text
vethaa4b5f0@if2
master br-92c88036c640
```

Conceptually:

```text
WSL Linux bridge
        ↓
host veth
        ||
        || veth pair
        ||
kind node eth0
172.21.0.2
```

So there are multiple nested network namespaces:

```text
WSL
↔
kind node
↔
Pod
```

---

# 8. Cross-Node Routing

Worker1 had:

```text
10.244.1.0/24 via 172.21.0.4 dev eth0
```

Worker2 had:

```text
10.244.2.0/24 via 172.21.0.2 dev eth0
```

So traffic from:

```text
10.244.2.2
```

to:

```text
10.244.1.2
```

travels:

```text
Pod A
10.244.2.2
   ↓
Pod gateway
10.244.2.1
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

# 9. Kubernetes Networking Components

The cluster contains:

```text
kindnet
kube-proxy
CoreDNS
```

Useful mental separation:

```text
kindnet
→ Pod networking / Pod CIDR connectivity

kube-proxy
→ Kubernetes Service dataplane

CoreDNS
→ cluster DNS
```

---

# 10. Backend Deployment

A three-replica Nginx Deployment was created:

```bash
kubectl create deployment backend \
  --image=nginx:alpine \
  --replicas=3
```

Observed backend Pods:

```text
10.244.1.5   devops-lab-worker2
10.244.2.5   devops-lab-worker
10.244.2.6   devops-lab-worker
```

A client Pod was also used:

```text
client
10.244.1.4
devops-lab-worker2
```

---

# 11. ClusterIP Service

The Service:

```text
backend-service
ClusterIP: 10.96.63.74
Port: 80
```

selected Pods using:

```text
app=backend
```

The important relationship is:

```text
Deployment
   ↓ creates Pods with labels

Pods:
app=backend

Service:
selector app=backend
   ↓
EndpointSlice
   ↓
backend Pod IPs
```

The Service does not attach directly to a Deployment.

It selects Pods.

---

# 12. EndpointSlice

Command:

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=backend-service \
  -o wide
```

Observed:

```text
10.244.2.5
10.244.1.5
10.244.2.6
```

EndpointSlices represent the actual destinations behind the Service.

---

# 13. Service DNS Resolution

From the client:

```bash
kubectl exec -it client -- getent hosts backend-service
```

returned:

```text
10.96.63.74
```

So:

```text
backend-service
→ 10.96.63.74
```

CoreDNS resolves the Service name to the Service ClusterIP.

CoreDNS does not choose the backend Pod for a normal ClusterIP Service.

---

# 14. kube-proxy Mode

The kube-proxy ConfigMap showed:

```text
mode: iptables
```

So in this cluster:

```text
kube-proxy
→ programs iptables rules
```

The packets themselves are handled by the Linux networking stack and iptables.

---

# 15. ClusterIP Is Virtual

Commands:

```bash
ip addr | grep 10.96.63.74
```

and:

```bash
docker exec devops-lab-worker ip addr | grep 10.96.63.74
```

returned nothing.

This confirms:

```text
10.96.63.74
```

is not assigned to a normal network interface.

It is a virtual Service IP implemented by kube-proxy rules.

---

# 16. ClusterIP iptables Entry

Observed:

```text
-A KUBE-SERVICES \
-d 10.96.63.74/32 \
-p tcp \
--dport 80 \
-j KUBE-SVC-ZZAJ2COS27FT6J6V
```

Meaning:

```text
traffic to 10.96.63.74:80
→ jump to Service-specific chain
```

---

# 17. Service Load Balancing

The Service chain contained:

```text
10.244.1.5:80 → probability ~1/3
10.244.2.5:80 → probability 1/2 of remaining traffic
10.244.2.6:80 → final fallback
```

Observed:

```text
-A KUBE-SVC-... -> 10.244.1.5:80 probability 0.333...
-A KUBE-SVC-... -> 10.244.2.5:80 probability 0.500...
-A KUBE-SVC-... -> 10.244.2.6:80
```

Because iptables evaluates rules sequentially, the effective distribution is approximately:

```text
10.244.1.5 → 33%
10.244.2.5 → 33%
10.244.2.6 → 33%
```

---

# 18. Endpoint Chains

Each endpoint had a `KUBE-SEP-*` chain.

Example:

```text
-A KUBE-SEP-LDPIAU4WTN36N2PH \
-p tcp \
-j DNAT --to-destination 10.244.2.6:80
```

This performs:

```text
10.96.63.74:80
→ DNAT
→ 10.244.2.6:80
```

So the Service path is:

```text
client
   ↓
ClusterIP
   ↓
KUBE-SERVICES
   ↓
KUBE-SVC
   ↓
KUBE-SEP
   ↓
DNAT
   ↓
Pod IP
```

---

# 19. Conntrack Observation

Repeated requests showed entries such as:

```text
src=10.244.1.4
dst=10.96.63.74
dport=80

reply:
src=10.244.2.6
dst=10.244.1.4
sport=80
```

This proves that the original connection targeted:

```text
10.96.63.74:80
```

while the real backend was:

```text
10.244.2.6:80
```

---

# 20. ClusterIP Packet Path

One observed path:

```text
client Pod
10.244.1.4
   ↓
backend-service
10.96.63.74:80
   ↓
iptables Service rules
   ↓
DNAT
   ↓
10.244.2.6:80
   ↓
worker2 routing
   ↓
worker1
   ↓
backend Pod
```

This gives an important separation:

```text
DNS
→ Service name to ClusterIP

kube-proxy / iptables
→ ClusterIP to endpoint Pod IP

Linux / CNI routing
→ Pod IP to correct node
```

---

# 21. KUBE-MARK-MASQ

Observed rules included:

```text
-j KUBE-MARK-MASQ
```

This does not immediately perform SNAT.

It means:

```text
mark this packet
→ masquerade/SNAT it later
```

Mental model:

```text
MARK NOW
SNAT LATER
```

This is used when Kubernetes needs return traffic to pass back through the same NAT path.

---

# 22. Hairpin Traffic

Endpoint chains also contained rules such as:

```text
-s 10.244.2.6/32
-j KUBE-MARK-MASQ
```

This handles cases where:

```text
Pod
→ Service
→ same Pod selected as backend
```

Without special handling, return traffic may bypass the expected NAT path.

Masquerading helps keep the connection symmetric through conntrack.

---

# 23. Failure Lab 1 — Wrong Service Selector

The Service selector was deliberately changed to:

```text
app=broken-backend
```

while Pods still had:

```text
app=backend
```

Result:

```text
Service exists
DNS works
ClusterIP exists
EndpointSlice has no usable endpoints
request fails
```

This proves:

```text
Service selector
must match
Pod labels
```

---

# 24. Connection Refused Did Not Mean Missing Listener

The failed request returned:

```text
curl: (7)
Connection refused
```

At first this can look like:

```text
nothing is listening on the target port
```

But inspecting iptables showed:

```text
-j REJECT
```

So the actual cause was:

```text
Service had no endpoints
→ kube-proxy installed REJECT behavior
→ client received immediate refusal
```

Important lesson:

```text
Connection refused
≠ automatically "application is not listening"

Connection refused
= something actively rejected the connection
```

Possible causes include:

```text
no application listener
iptables REJECT
Service with no endpoints
firewall reject
proxy reject
```

---

# 25. Failure Lab 2 — Wrong targetPort

The Service was changed from:

```text
port: 80
targetPort: 80
```

to:

```text
port: 80
targetPort: 81
```

The EndpointSlice still existed, but now showed:

```text
10.244.2.5:81
10.244.1.5:81
10.244.2.6:81
```

iptables showed:

```text
DNAT --to-destination 10.244.x.x:81
```

The client again received:

```text
Connection refused
```

but this failure was different.

Path:

```text
DNS works
Service exists
Endpoints exist
DNAT works
traffic reaches Pod:81
nothing listening on 81
TCP RST
```

So two failures can have the same client symptom:

```text
Connection refused
```

but different root causes.

---

# 26. Service Troubleshooting Sequence

A useful sequence is:

```text
1. Does DNS resolve?

2. Does the Service exist?

3. Does the EndpointSlice contain endpoints?

4. Does the Service selector match Pod labels?

5. Is targetPort correct?

6. Is kube-proxy / Service NAT working?

7. Can the node route to the selected Pod?

8. Is the application listening?
```

---

# 27. Kubernetes DNS

Inside the client Pod:

```bash
kubectl exec client -- cat /etc/resolv.conf
```

Observed:

```text
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

The DNS nameserver is:

```text
10.96.0.10
```

---

# 28. kube-dns Service

Command:

```bash
kubectl get svc -n kube-system kube-dns -o wide
```

Observed:

```text
ClusterIP: 10.96.0.10
Ports:
53/UDP
53/TCP
9153/TCP
```

Important distinction:

```text
10.96.0.10
= kube-dns Service ClusterIP

10.244.0.2
10.244.0.3
= actual CoreDNS Pods
```

---

# 29. DNS Service Path

CoreDNS Pods:

```text
10.244.0.2
10.244.0.3
```

The DNS query path is:

```text
client Pod
   ↓
nameserver 10.96.0.10:53
   ↓
kube-dns ClusterIP
   ↓
kube-proxy / iptables
   ↓
DNAT
   ↓
10.244.0.2:53
or
10.244.0.3:53
   ↓
CoreDNS
```

So kube-proxy is not the DNS server.

It programs the Service NAT rules that allow the kube-dns Service IP to reach the CoreDNS Pods.

---

# 30. CoreDNS Resolution

For:

```text
backend-service
```

CoreDNS returns:

```text
10.96.63.74
```

because the Service exists in the Kubernetes API.

Conceptually:

```text
Kubernetes API
   ↓
Service object

backend-service
namespace: default
clusterIP: 10.96.63.74
   ↓
CoreDNS
   ↓
DNS answer
```

---

# 31. Kubernetes Service FQDN

The full Service name is:

```text
backend-service.default.svc.cluster.local
```

Structure:

```text
<service>.<namespace>.svc.cluster.local
```

In this case:

```text
backend-service
→ Service name

default
→ namespace

svc
→ Kubernetes Service DNS zone

cluster.local
→ cluster DNS domain
```

---

# 32. Short DNS Names

These all resolved:

```text
backend-service
backend-service.default
backend-service.default.svc.cluster.local
```

The search domains in `/etc/resolv.conf` allow short names to work.

---

# 33. dig vs getent

This worked:

```bash
getent hosts backend-service
```

But plain:

```bash
dig backend-service
```

returned NXDOMAIN.

Reason:

```text
getent
→ uses normal libc/NSS resolver behavior
→ search domains applied

dig backend-service
→ usually queries the name as written

dig +search backend-service
→ applies search domains
```

So:

```text
getent works
dig fails
```

does not automatically mean CoreDNS is broken.

---

# 34. Internal vs External DNS

Internal Service lookup:

```text
backend-service
→ CoreDNS answers from Kubernetes data
```

External lookup:

```text
google.com
→ CoreDNS forwards query upstream
```

Observed:

```text
SERVER: 10.96.0.10#53
```

for both types of DNS query.

So the Pod always talks to:

```text
10.96.0.10
```

and CoreDNS decides whether to answer internally or forward upstream.

---

# 35. Two Separate DNAT Events

When an application runs:

```bash
curl http://backend-service
```

there can be two separate Service DNAT operations.

First:

```text
DNS query

10.96.0.10:53
→ DNAT
→ CoreDNS Pod:53
```

CoreDNS returns:

```text
backend-service = 10.96.63.74
```

Then the HTTP connection starts:

```text
10.96.63.74:80
→ DNAT
→ backend Pod:80
```

So:

```text
DNAT #1
kube-dns ClusterIP
→ CoreDNS Pod

DNAT #2
backend ClusterIP
→ application Pod
```

---

# 36. NodePort Service

A second Service was created:

```text
backend-nodeport
```

Observed:

```text
ClusterIP: 10.96.151.62
Service port: 80
NodePort: 32559
```

Command:

```bash
kubectl get svc backend-nodeport -o wide
```

Output included:

```text
80:32559/TCP
```

Meaning:

```text
Service port = 80
NodePort = 32559
```

---

# 37. NodePort Entry Points

Node IPs:

```text
172.21.0.2
172.21.0.3
172.21.0.4
```

So with the default NodePort behavior, the Service can be accessed using:

```text
172.21.0.2:32559
172.21.0.3:32559
172.21.0.4:32559
```

---

# 38. NodePort with externalTrafficPolicy: Cluster

Default:

```text
externalTrafficPolicy: Cluster
```

Meaning:

```text
request enters any node
→ backend can be on any node
```

Example:

```text
request enters worker1
172.21.0.2:32559
```

but kube-proxy may select:

```text
10.244.1.5
```

which is on worker2.

So:

```text
worker1
   ↓
Service rules
   ↓
backend on worker2
```

is valid.

---

# 39. NodePort Conntrack

Requests to:

```text
172.21.0.2:32559
```

showed backend responses from:

```text
10.244.2.5
10.244.2.6
10.244.1.5
```

This proved that one NodePort entry point was load-balancing across Pods on multiple nodes.

---

# 40. NodePort iptables Chain

Observed:

```text
KUBE-NODEPORTS
   ↓
KUBE-EXT-QYRALRIZFZSY4I4Q
   ↓
KUBE-MARK-MASQ
   ↓
KUBE-SVC-QYRALRIZFZSY4I4Q
   ↓
KUBE-SEP
   ↓
DNAT to PodIP:80
```

Important rule:

```text
-A KUBE-EXT-QYRALRIZFZSY4I4Q \
-j KUBE-SVC-QYRALRIZFZSY4I4Q
```

So NodePort reuses the normal Service backend selection chain.

---

# 41. ClusterIP vs NodePort Path

ClusterIP:

```text
ClusterIP:80
   ↓
KUBE-SERVICES
   ↓
KUBE-SVC
   ↓
KUBE-SEP
   ↓
Pod
```

NodePort:

```text
NodeIP:NodePort
   ↓
KUBE-NODEPORTS
   ↓
KUBE-EXT
   ↓
KUBE-SVC
   ↓
KUBE-SEP
   ↓
Pod
```

NodePort adds an external entry path.

---

# 42. Why NodePort Uses Masquerading

With:

```text
externalTrafficPolicy: Cluster
```

NodePort traffic can be sent to a Pod on another node.

To maintain a consistent return path, kube-proxy may mark external traffic for SNAT/MASQUERADE.

Trade-off:

```text
Cluster
+ any backend in cluster can serve traffic
- original external source IP may be hidden
```

---

# 43. externalTrafficPolicy: Local

The Service was changed to:

```text
externalTrafficPolicy: Local
```

Pod placement:

```text
worker1
172.21.0.2
→ 10.244.2.5
→ 10.244.2.6

worker2
172.21.0.4
→ 10.244.1.5

control-plane
172.21.0.3
→ no backend
```

Test results:

```text
172.21.0.2:32559 → success
172.21.0.4:32559 → success
172.21.0.3:32559 → timeout
```

---

# 44. Local Endpoint Chain

With `externalTrafficPolicy: Local`, kube-proxy created:

```text
KUBE-SVL-...
```

On worker1:

```text
KUBE-SVL
→ 10.244.2.5
→ 10.244.2.6
```

On worker2:

```text
KUBE-SVL
→ 10.244.1.5
```

So:

```text
KUBE-SVC
→ all endpoints

KUBE-SVL
→ local endpoints only
```

---

# 45. NodePort Local Policy Path

With:

```text
externalTrafficPolicy: Local
```

the external path becomes:

```text
external client
   ↓
NodeIP:NodePort
   ↓
KUBE-NODEPORTS
   ↓
KUBE-EXT
   ↓
KUBE-SVL
   ↓
local backend only
```

Benefits:

```text
original client IP can be preserved
cross-node hop avoided
```

Trade-off:

```text
node without local backend cannot serve the external request
```

---

# 46. Cluster vs Local

## Cluster

```text
externalTrafficPolicy: Cluster
```

Behavior:

```text
any node
→ any backend Pod
```

Advantages:

```text
better availability
all healthy Pods usable
```

Disadvantages:

```text
possible extra cross-node hop
external client source IP may be SNATed
```

---

## Local

```text
externalTrafficPolicy: Local
```

Behavior:

```text
node
→ local backend Pods only
```

Advantages:

```text
preserves original external source IP
avoids extra cross-node forwarding
```

Disadvantages:

```text
node without local backend cannot serve request
```

---

# 47. Important Failure Symptoms

## Connection Refused

Possible causes:

```text
application not listening
wrong targetPort
iptables REJECT
Service with no endpoints
firewall actively rejecting
```

Do not assume:

```text
connection refused
=
application missing
```

---

## Timeout

Possible causes:

```text
DROP
missing return path
routing failure
firewall silently dropping
externalTrafficPolicy Local with no usable local endpoint
```

---

## DNS Failure

Possible causes:

```text
wrong Service name
wrong namespace
CoreDNS problem
kube-dns Service problem
DNS Service networking failure
```

---

## Service Exists but Request Fails

Check:

```text
EndpointSlice
selector
targetPort
iptables rules
Pod route
application listener
```

---

# 48. Troubleshooting Mental Model

Instead of memorizing every command, ask:

```text
At which layer did the packet stop?
```

A useful order:

```text
Application name
   ↓
DNS resolution
   ↓
Service object
   ↓
EndpointSlice
   ↓
Service NAT / kube-proxy
   ↓
Pod routing
   ↓
destination node
   ↓
destination Pod
   ↓
application port
```

---

# 49. Practical Troubleshooting Commands

## Pods

```bash
kubectl get pods -o wide
```

```bash
kubectl get pods --show-labels
```

---

## Nodes

```bash
kubectl get nodes -o wide
```

---

## Services

```bash
kubectl get svc
```

```bash
kubectl get svc <service> -o yaml
```

---

## EndpointSlices

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=<service> \
  -o wide
```

---

## DNS

```bash
kubectl exec <pod> -- cat /etc/resolv.conf
```

```bash
kubectl exec <pod> -- getent hosts <service>
```

```bash
kubectl exec <pod> -- dig +search <service>
```

---

## Pod networking

```bash
kubectl exec <pod> -- ip addr
```

```bash
kubectl exec <pod> -- ip route
```

---

## Node routing

```bash
docker exec <kind-node> ip route
```

```bash
docker exec <kind-node> ip route get <pod-ip>
```

---

## Service NAT

```bash
docker exec <kind-node> \
  iptables-save -t nat
```

---

## Service-specific rules

```bash
docker exec <kind-node> \
  iptables-save -t nat | grep <service-name>
```

---

## Conntrack

```bash
docker exec <kind-node> conntrack -L
```

---

# 50. Final Mental Model

For internal application traffic:

```text
application
   ↓
Service DNS name
   ↓
CoreDNS
   ↓
ClusterIP
   ↓
kube-proxy / iptables
   ↓
DNAT
   ↓
Pod IP
   ↓
Linux/CNI routing
   ↓
application container
```

For NodePort traffic:

```text
external client
   ↓
NodeIP:NodePort
   ↓
KUBE-NODEPORTS
   ↓
Service dataplane
   ↓
DNAT
   ↓
backend Pod
```

---

# 51. What Must Be Remembered

Do not memorize chain hashes such as:

```text
KUBE-SVC-ZZAJ2COS27FT6J6V
KUBE-SEP-LDPIAU4WTN36N2PH
```

Remember the roles:

```text
KUBE-SERVICES
→ Service entry matching

KUBE-SVC
→ Service backend selection

KUBE-SEP
→ actual endpoint / DNAT

KUBE-NODEPORTS
→ NodePort entry

KUBE-EXT
→ external traffic handling

KUBE-SVL
→ local endpoints only

KUBE-MARK-MASQ
→ mark traffic for later SNAT/MASQUERADE
```

The exact generated chain names can always be inspected.

---

# 52. Core Troubleshooting Knowledge

The most useful understanding from this lab is:

```text
DNS works
does not mean
Service works
```

```text
Service exists
does not mean
endpoints exist
```

```text
Endpoints exist
does not mean
targetPort is correct
```

```text
Connection refused
does not always mean
application is not listening
```

```text
NodePort exists
does not mean
every node can serve traffic when externalTrafficPolicy=Local
```

The goal is to identify which layer failed instead of guessing from the final error message.