# Kubernetes Networking – Quick Notes

## 1. Main Networking Layers

```text
WSL host
   ↓
kind node container
   ↓
Pod network namespace
```

In kind:

```text
Node network:
172.21.0.0/16

Pod network:
10.244.0.0/16

Service network:
10.96.0.0/12
```

These are different networks serving different purposes.

---

## 2. Pod Network Namespace

Each normal Pod gets its own Linux network namespace.

Example:

```text
pvc-demo
Pod IP: 10.244.1.4
```

Inside Pod network namespace:

```text
lo
eth0 = 10.244.1.4
routes
```

Pod network namespace ID:

```text
net:[...]
```

is different from the node network namespace.

---

## 3. veth Pair

A veth pair is like a virtual Ethernet cable.

```text
Pod netns                     Node netns

eth0  <===================>  veth510f511c
10.244.1.4                    10.244.1.1
```

- Pod side = `eth0`
- Node side = `veth...`
- They are the two ends of the same pair.

The node route:

```text
10.244.1.4 dev veth510f511c
```

means:

```text
To reach Pod 10.244.1.4
→ send traffic through that veth
```

---

## 4. Same-Node Pod Traffic

Example:

```text
Pod A = 10.244.1.4
Pod B = 10.244.1.2
```

Flow:

```text
Pod A eth0
 ↓
node-side veth A
 ↓
node routing table
 ↓
node-side veth B
 ↓
Pod B eth0
```

Node routes:

```text
10.244.1.2 dev vethA
10.244.1.4 dev vethB
```

The node kernel routes traffic between the Pod network namespaces.

---

## 5. Cross-Node Pod Traffic

Example worker2 routes:

```text
10.244.0.0/24 via 172.21.0.2
10.244.2.0/24 via 172.21.0.4
```

Therefore:

```text
Pod
 ↓
veth
 ↓
source node routing
 ↓
node eth0
 ↓
destination node eth0
 ↓
destination node routing
 ↓
destination veth
 ↓
Pod
```

Example:

```text
10.244.1.4
 ↓
worker2
 ↓
172.21.0.2
 ↓
10.244.0.x Pod
```

---

## 6. Node Default Route

Example:

```text
default via 172.21.0.1 dev eth0
```

`172.21.0.1` is the Docker/kind network gateway.

Used when no more-specific route exists:

```text
Pod
 ↓
worker node
 ↓
172.21.0.1
 ↓
WSL / host networking
 ↓
external network
```

---

## 7. Pod Routing

Example Pod routing:

```text
default via 10.244.1.1 dev eth0
```

So traffic leaving the Pod goes:

```text
Pod
 ↓
eth0
 ↓
10.244.1.1
 ↓
node-side veth
 ↓
node routing
```

---

# Pod vs Kubernetes Namespace

Do not confuse:

```text
Kubernetes namespace
→ default
→ kube-system
→ ingress-nginx
```

with:

```text
Linux network namespace
→ interfaces
→ IP addresses
→ routes
→ neighbour table
```

They are completely different concepts.

---

# Pod vs Container Networking

In Kubernetes:

```text
1 Pod
→ 1 shared network namespace
→ 1 Pod IP
```

Multiple containers inside the Pod normally share:

```text
Pod IP
network interfaces
routes
localhost
ports
```

Example:

```text
Pod IP: 10.244.1.10

Container A
Container B
     ↓
same network namespace
```

Containers can communicate through:

```text
localhost
```

---

## Container-Specific Resources

Containers normally still have their own:

```text
root filesystem
mount namespace
image
environment
process namespace
```

Volumes can be explicitly shared.

---

# Pod Sandbox / CRI

Creation flow:

```text
API Server
 ↓
Scheduler
 ↓
Kubelet
 ↓
CRI
 ↓
RunPodSandbox
 ↓
Pod sandbox
 ↓
network namespace
 ↓
CNI configures networking
 ↓
CreateContainer
 ↓
StartContainer
```

Important order:

```text
Sandbox
→ network
→ containers
```

Containers join the already prepared Pod network namespace.

---

## IDs

```text
Kubernetes Pod name
→ pvc-demo

Kubernetes Pod UID
→ metadata.uid

CRI PodSandbox ID
→ shown as "POD ID" in crictl pods

Container ID
→ separate runtime ID
```

`crictl pods` really shows PodSandbox IDs.

---

# DNS – CoreDNS

Inside containers:

```text
/etc/resolv.conf
```

normally contains:

```text
nameserver 10.96.0.10
```

That is the `kube-dns` Service.

Flow:

```text
application asks for service name
 ↓
/etc/resolv.conf
 ↓
10.96.0.10
 ↓
kube-dns Service
 ↓
CoreDNS Pod
 ↓
Service name resolved to Service IP
```

Example:

```text
my-api.default.svc.cluster.local
 ↓
10.96.50.20
```

---

# kube-proxy

kube-proxy handles Service traffic.

Example:

```text
Service IP:
10.96.50.20
```

Backend Pods:

```text
10.244.1.4
10.244.2.5
10.244.0.8
```

Conceptually:

```text
10.96.50.20
 ↓
Service dataplane rules
 ↓
one backend Pod IP
```

kube-proxy watches:

```text
Services
EndpointSlices
```

and programs node networking rules.

---

# Direct Pod IP vs Service IP

If destination is directly:

```text
10.244.x.x
```

then:

```text
No CoreDNS required
No Service required
No kube-proxy required

→ normal CNI/Linux routing
```

If destination is:

```text
10.96.x.x
```

then Service handling is involved.

---

# Complete Request Flow

Using a Service name:

```text
Application
 ↓
service-name
 ↓
CoreDNS
 ↓
Service IP
 ↓
kube-proxy / Service dataplane
 ↓
Endpoint Pod IP
 ↓
Linux routing
 ↓
node eth0 / veth
 ↓
destination Pod
```

Easy memory:

```text
CoreDNS
= Name → Service IP

kube-proxy
= Service IP → Pod endpoint

CNI + Linux routing
= Pod IP → actual Pod
```

---

# Main Mental Model

```text
Container
 ↓
Pod network namespace
 ↓
Pod eth0
 ↓
veth
 ↓
Node network namespace
 ↓
Linux routing
 ↓
local Pod / remote node / external network
```

Nothing is magic:

Kubernetes networking is mostly Linux namespaces, veth pairs, routes, packet-processing rules, and controllers automating their configuration.