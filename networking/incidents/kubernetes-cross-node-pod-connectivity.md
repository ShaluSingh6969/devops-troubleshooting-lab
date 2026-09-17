# Incident — Kubernetes Cross-Node Pod Connectivity

## Scenario

Two Pods run on different Kubernetes worker nodes.

```text
Pod A
IP: 10.244.2.2
Node: devops-lab-worker
```

```text
Pod B
IP: 10.244.1.2
Node: devops-lab-worker2
```

Goal:

```text
Pod A
→
Pod B
```

using direct Pod IP connectivity.

---

## Validation

Command:

```bash
kubectl get pods -o wide
```

confirmed:

```text
pod-a   10.244.2.2   devops-lab-worker
pod-b   10.244.1.2   devops-lab-worker2
```

---

## Connectivity Test

From Pod A:

```bash
kubectl exec -it pod-a -- ping -c 3 10.244.1.2
```

Result:

```text
3 packets transmitted
3 packets received
0% packet loss
```

Cross-node Pod networking was working.

---

## Pod A Routing

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

Pod A sends non-local traffic to:

```text
10.244.2.1
```

---

## Node-Side Interface

On `devops-lab-worker`:

```text
vetha80f48b3@if2
inet 10.244.2.1/32
```

This is the node-side interface connected to Pod A.

Conceptually:

```text
Pod A
10.244.2.2
   ↓
eth0
   ↓
veth pair
   ↓
vetha80f48b3
10.244.2.1
```

---

## Worker1 Routing

Worker1 had:

```text
10.244.1.0/24 via 172.21.0.4 dev eth0
```

Since Pod B is:

```text
10.244.1.2
```

worker1 chooses:

```text
next hop = 172.21.0.4
```

which is worker2.

---

## Worker2 Routing

Worker2 had:

```text
10.244.2.0/24 via 172.21.0.2 dev eth0
```

This provides the return path toward Pod A.

---

## Complete Packet Path

```text
Pod A
10.244.2.2
   ↓
gateway 10.244.2.1
   ↓
worker1 routing
   ↓
10.244.1.0/24 via 172.21.0.4
   ↓
worker2
172.21.0.4
   ↓
Pod B
10.244.1.2
```

---

## Troubleshooting Pattern

If same-node Pod traffic works but cross-node traffic fails:

```text
Check destination Pod IP
        ↓
Check destination Node
        ↓
Check Node-to-Node connectivity
        ↓
Check source Node route for destination Pod CIDR
        ↓
Check CNI health
        ↓
Check NetworkPolicy
        ↓
Check destination application port
```

Useful commands:

```bash
kubectl get pods -o wide
```

```bash
kubectl get nodes -o wide
```

```bash
ip route
```

```bash
ip route get <pod-ip>
```

```bash
kubectl exec -it <pod> -- ping <pod-ip>
```

---

## Key Lesson

Cross-node Pod communication depends on:

```text
Pod network namespace
+
veth connectivity
+
correct node routing
+
working CNI
+
reachable destination node
```

If local Pod networking works but remote Pod networking fails, the node-to-node/CNI routing path should be investigated early.