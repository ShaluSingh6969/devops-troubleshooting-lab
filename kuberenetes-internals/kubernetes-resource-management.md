# Kubernetes Resource Management

## Requests and Limits

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "300m"
    memory: "128Mi"
```

### Requests
Used mainly by the scheduler.

They answer:

> How much resource should Kubernetes assume this Pod needs?

A Pod may stay `Pending` if no node has enough allocatable resources for its requests.

### Limits
Used mainly at runtime.

- CPU limit exceeded -> container is throttled
- Memory limit exceeded -> process may be killed with `OOMKilled`

---

## QoS

QoS = Quality of Service.

Classes:

```text
Guaranteed
Burstable
BestEffort
```

### Guaranteed

CPU and memory requests equal limits for every container.

### Burstable

Some requests/limits exist, but they are not fully equal.

### BestEffort

No requests or limits are configured.

---

## Node Pressure

Kubelet monitors node-level resources such as:

```text
memory.available
nodefs.available
nodefs.inodesFree
imagefs.available
PID availability
```

Example:

```yaml
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
```

These represent minimum available-resource thresholds.

Example:

```text
memory.available < 100Mi
→ kubelet may start reclaiming resources / evicting Pods
```

---

## OOMKilled vs Eviction

```text
Container exceeds memory limit
→ kernel/cgroup OOM
→ container killed
→ OOMKilled
```

```text
Node runs low on resources
→ kubelet eviction manager
→ Pod may be evicted
```

---

## Image and Filesystem Pressure

Running containers still depend on their image layers.

```text
container filesystem
=
read-only image layers
+
container writable layer
```

Image garbage collection removes unused images.

Pod eviction may reclaim:

```text
container writable data
logs
emptyDir data
ephemeral storage
```

and can make previously used images eligible for later garbage collection.

---

## Monitoring

Monitor both infrastructure and Kubernetes state.

Useful areas:

```text
Node memory available
Node filesystem available
Free inodes
Image filesystem usage
MemoryPressure
DiskPressure
PIDPressure
Pod Pending
Pod Evicted
OOMKilled
Deployment unavailable replicas
```

Alert before kubelet eviction thresholds are reached.

Example:

```text
warning:
memory.available < 1Gi

critical:
memory.available < 500Mi

hard eviction threshold:
memory.available < 100Mi
```

---

## Troubleshooting Flow

```text
Deployment unhealthy
↓
Pod Pending / Evicted / OOMKilled
↓
kubectl describe pod
↓
check events
↓
check requests/limits
↓
check node allocatable resources
↓
check node pressure conditions
↓
check monitoring metrics
```