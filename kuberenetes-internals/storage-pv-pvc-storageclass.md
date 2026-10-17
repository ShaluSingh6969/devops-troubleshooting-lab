# Kubernetes Storage – Quick Notes

## 1. Storage Flow

```text
Pod
 ↓
volumeMount
 ↓
PVC
 ↓
PV
 ↓
StorageClass / CSI Provisioner
 ↓
Actual Storage
```

- **volumeMount** → where storage appears inside the container.
- **PVC (PersistentVolumeClaim)** → application requests storage.
- **PV (PersistentVolume)** → Kubernetes resource representing actual storage.
- **StorageClass** → defines how storage should be provisioned.
- **CSI (Container Storage Interface)** → driver used by Kubernetes to communicate with storage systems.

---

## 2. Dynamic Provisioning

Our StorageClass:

```text
standard
Provisioner: rancher.io/local-path
ReclaimPolicy: Delete
VolumeBindingMode: WaitForFirstConsumer
```

Initially:

```text
PVC created
→ PVC = Pending
→ no PV
```

Because:

```text
WaitForFirstConsumer
```

means Kubernetes waits until a Pod actually requests the PVC.

Then:

```text
Pod uses PVC
 ↓
Scheduler selects node
 ↓
Provisioner creates storage
 ↓
PV created
 ↓
PVC becomes Bound
 ↓
Pod starts
```

---

## 3. What We Observed

PVC:

```text
app-data-pvc
1Gi
RWO
StorageClass: standard
```

Pod was scheduled to:

```text
devops-lab-worker2
```

Kubernetes dynamically created a PV:

```text
pvc-<uuid>
```

PV source:

```text
HostPath:
/var/local-path-provisioner/pvc-...
```

Node affinity:

```text
devops-lab-worker2
```

Therefore this particular storage is **node-local**.

---

## 4. kind Storage Path

Important:

In kind, the Kubernetes node itself is a Docker container.

So:

```text
WSL host
 ↓
Docker volume
 ↓
kind worker /var
 ↓
/var/local-path-provisioner/pvc-...
 ↓
Pod /data
```

Example:

```text
WSL:
/var/lib/docker/volumes/<volume-id>/_data

        ↓ mounted as

kind node:
/var

        ↓

PV storage:
/var/local-path-provisioner/pvc-...

        ↓ mounted into

Pod:
/data
```

In a normal cluster without kind, the node path would exist on the **actual worker node**.

---

## 5. Local vs Network Storage

### Local storage

```text
worker-1
 ↓
local disk
 ↓
PV
```

Storage is tied to a particular node.

If the Pod moves to another node, that node may not have access to the data.

### Network / Cloud Storage

```text
worker-1 ─┐
worker-2 ─┼→ external/network storage
worker-3 ─┘
```

Examples:

- EBS
- EFS
- NFS
- Ceph

Scheduling still depends on storage constraints such as:

- availability zone
- access mode
- CSI driver
- topology

---

# 6. Access Modes

## RWO — ReadWriteOnce

```text
Read/write from one NODE
```

Important:

```text
RWO ≠ one Pod
```

Multiple Pods may potentially use it if they are on the same node.

---

## ROX — ReadOnlyMany

```text
Many nodes
↓
read-only access
```

---

## RWX — ReadWriteMany

```text
worker-1 → Pod
worker-2 → Pod
worker-3 → Pod
       ↓
shared storage
```

Multiple nodes can read and write.

Common with:

- NFS
- EFS
- CephFS

---

## RWOP — ReadWriteOncePod

```text
One Pod only
```

Stricter than RWO.

---

## Easy Memory

```text
RWO  = one Node writes
ROX  = many Nodes read
RWX  = many Nodes write
RWOP = one Pod writes
```

Actual support depends on the storage backend / CSI driver.

---

# 7. Reclaim Policy

Controls what happens to storage when the **PVC is deleted**.

## Delete

```text
Delete PVC
 ↓
Delete PV
 ↓
Delete underlying storage
```

Common for dynamically provisioned storage.

Our StorageClass uses:

```text
ReclaimPolicy: Delete
```

---

## Retain

```text
Delete PVC
 ↓
PV becomes Released
 ↓
Underlying data remains
```

Administrator must manually recover, reuse, or delete the storage.

Useful when accidental deletion must not automatically destroy important data.

---

# 8. Persistence Comparison

### emptyDir

```text
Container restart → survives
Pod deletion      → data lost
```

### PVC / PV

```text
Container restart → survives
Pod deletion      → survives
Pod recreation    → survives
PVC deletion      → depends on ReclaimPolicy
```

---

# 9. WaitForFirstConsumer

Instead of provisioning storage immediately:

```text
PVC
 ↓
Pending
```

Kubernetes waits for:

```text
Pod → PVC
```

Then it knows where the Pod should run and can provision compatible storage.

Useful for storage with topology constraints such as:

```text
Availability Zones
Node-local storage
Cloud block volumes
```

---

# 10. Main Mental Model

Nothing magical is happening.

Kubernetes eventually creates normal storage mounts:

```text
Actual storage
 ↓
Node filesystem
 ↓
kubelet
 ↓
container mount namespace
 ↓
volumeMount
 ↓
application path
```

Kubernetes mainly automates:

```text
requesting
provisioning
binding
scheduling
mounting
lifecycle management
```

of that storage.