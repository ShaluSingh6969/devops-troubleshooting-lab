# Kubernetes Storage Internals

## Goal

This session focused on tracing Kubernetes storage all the way down to Linux filesystem mounts and understanding how Kubernetes storage abstractions map to Docker, containerd, kubelet, OverlayFS, mount namespaces, and the underlying host filesystem.

Topics covered:

- `emptyDir`
- ConfigMap volumes
- projected ConfigMap filesystem layout
- Pod vs container storage lifetime
- kind node storage
- Docker bind mounts vs Docker volumes
- kubelet storage paths
- containerd OverlayFS snapshots
- tracing storage with `/proc/self/mountinfo`
- tracing storage with `findmnt`
- lowerdir / upperdir / merged view
- cleanup and garbage collection
- PV / PVC / StorageClass / CSI
- CRI / containerd / shim relationship

---

# 1. Kubernetes `emptyDir`

Example Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "sleep 10000"]
      volumeMounts:
        - name: app-data
          mountPath: /data

  volumes:
    - name: app-data
      emptyDir: {}
```

Apply:

```bash
kubectl apply -f volume-demo.yaml
```

Write data:

```bash
kubectl exec -it volume-demo -- sh
```

Inside:

```sh
echo "hello from kubernetes" > /data/test.txt
cat /data/test.txt
```

---

# 2. `emptyDir` Lifetime

We proved that `emptyDir` belongs to the Pod lifecycle, not to an individual container lifecycle.

Mental model:

```text
container restart
→ same Pod
→ same emptyDir
→ data survives
```

But:

```text
Pod deletion
→ Pod recreated
→ new emptyDir
→ old data disappears
```

Important rule:

```text
emptyDir lifetime = Pod lifetime
```

---

# 3. Restarting a Container Through CRI

First identify the Pod and node:

```bash
kubectl get pod volume-demo -o wide
```

Check the container ID:

```bash
kubectl get pod volume-demo \
  -o jsonpath='{.metadata.uid}{"\n"}{.status.containerStatuses[0].containerID}{"\n"}{.status.containerStatuses[0].restartCount}{"\n"}{.status.containerStatuses[0].state}{"\n"}'
```

Example:

```text
containerd://53aeb282f2a73...
```

Because this is a kind cluster, each Kubernetes node is itself a Docker container.

Check containers inside the correct worker node:

```bash
docker exec devops-lab-worker crictl ps
```

Stop the workload container:

```bash
docker exec devops-lab-worker \
  crictl stop <CONTAINER_ID>
```

This triggered:

```text
container stops
    ↓
kubelet notices desired state is not satisfied
    ↓
restartPolicy = Always
    ↓
kubelet asks containerd to create/start replacement
    ↓
new container
    ↓
same Pod
    ↓
emptyDir survives
```

Important distinction:

```text
kubectl
→ talks to Kubernetes API

crictl
→ talks directly to CRI runtime on a specific node
```

---

# 4. kind Node Runtime Layout

In kind:

```text
WSL host
├── devops-lab-control-plane
│    └── containerd
│
└── devops-lab-worker
     └── containerd
          └── Kubernetes workload containers
```

Each node has its own runtime.

That means CRI commands must be executed on the node where the Pod is actually running.

---

# 5. ConfigMap as a Volume

Create ConfigMap:

```bash
kubectl create configmap app-config \
  --from-literal=app.properties='mode=production'
```

Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "sleep 10000"]
      volumeMounts:
        - name: config
          mountPath: /config

  volumes:
    - name: config
      configMap:
        name: app-config
```

Apply:

```bash
kubectl apply -f config-demo.yaml
```

Inspect:

```bash
kubectl exec config-demo -- ls -la /config
```

Observed structure:

```text
..2026_10_14_08_56_36.3302416812/
..data -> ..2026_10_14_08_56_36.3302416812
app.properties -> ..data/app.properties
```

---

# 6. Why ConfigMap Uses Symlinks

The effective structure is:

```text
/config
├── ..2026_10_14_08_56_36.../
│   └── app.properties
│
├── ..data -> ..2026_10_14_08_56_36...
│
└── app.properties -> ..data/app.properties
```

The application reads:

```text
/config/app.properties
```

which resolves to:

```text
/config/app.properties
        ↓
/config/..data/app.properties
        ↓
/config/..2026_.../app.properties
```

Kubernetes can update ConfigMap contents by creating a new versioned directory and switching the `..data` symlink.

Mental model:

```text
ConfigMap object
    ↓
kubelet materializes files
    ↓
versioned directory
    ↓
..data symlink
    ↓
container-visible file
```

---

# 7. ConfigMap Is a Projected Filesystem View

A ConfigMap volume is not simply:

```text
one existing host file
→ bind mount
→ container
```

Instead:

```text
Kubernetes API data
        ↓
kubelet
        ↓
generated files + symlinks
        ↓
mounted into Pod
```

Example ConfigMap:

```yaml
data:
  app.properties: mode=production
```

becomes:

```text
/config/app.properties
```

inside the container.

The bytes still ultimately live on node storage.

---

# 8. Tracing `/config` From the Pod

From inside the Pod:

```bash
kubectl exec config-demo -- \
  cat /proc/self/mountinfo | grep ' /config '
```

Observed:

```text
2241 2233 8:48 /var/lib/docker/volumes/9f2155.../_data/lib/kubelet/pods/<POD_UID>/volumes/kubernetes.io~configmap/config /config ro,relatime - ext4 /dev/sdd ...
```

Important fields:

```text
source:
 /var/lib/docker/volumes/.../_data/lib/kubelet/pods/.../config

target:
 /config
```

So `/config` is ultimately backed by node-side storage.

---

# 9. Docker Mounts Used by the kind Node

Inspect kind worker mounts:

```bash
docker inspect devops-lab-worker \
  --format '{{range .Mounts}}{{println .Type .Source "->" .Destination}}{{end}}'
```

Observed:

```text
bind /lib/modules -> /lib/modules
volume /var/lib/docker/volumes/9f2155.../_data -> /var
```

This means:

```text
WSL host:
/var/lib/docker/volumes/9f2155.../_data

        ↓ Docker volume

kind worker:
/var
```

Therefore:

```text
kind node:
/var/lib/kubelet/...

maps to:

WSL:
/var/lib/docker/volumes/9f2155.../_data/lib/kubelet/...
```

---

# 10. Bind Mount vs Docker Volume

## Bind mount

Example:

```text
bind /lib/modules -> /lib/modules
```

Source is an existing host path:

```text
WSL /lib/modules
        ↓
kind node /lib/modules
```

Docker does not own the source.

## Docker volume

Example:

```text
volume /var/lib/docker/volumes/.../_data -> /var
```

Docker manages the backing source:

```text
/var/lib/docker/volumes/<volume-id>/_data
        ↓
kind node /var
```

At Linux kernel level both become mounts.

Difference:

```text
bind mount
→ source path chosen externally

Docker volume
→ source path managed by Docker
```

---

# 11. Tracing Storage With `findmnt`

Inside the kind node:

```bash
findmnt -T \
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1/fs
```

Observed:

```text
TARGET  /var
SOURCE  /dev/sdd[/var/lib/docker/volumes/9f2155.../_data]
FSTYPE  ext4
```

Meaning:

```text
kind node path:
/var/lib/containerd/...

        ↓

belongs to:
/var

        ↓

which is backed by:
WSL Docker volume

        ↓

which lives on:
/dev/sdd

        ↓

filesystem:
ext4
```

Useful command:

```bash
findmnt -T <PATH>
```

Meaning:

```text
"Which filesystem or mount contains this path?"
```

---

# 12. Nested Runtime Architecture in kind

kind adds an extra runtime layer:

```text
WSL host
  ↓
Docker Engine
  ↓
kind worker container
  ↓
kubelet
  ↓
containerd
  ↓
Kubernetes workload containers
```

A real Kubernetes node is simpler:

```text
VM / physical node
  ↓
kubelet
  ↓
containerd
  ↓
Pod containers
```

---

# 13. containerd OverlayFS Inside kind

Observed:

```text
overlay on /run/containerd/io.containerd.runtime.v2.task/k8s.io/<id>/rootfs
type overlay (
  rw,
  lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1/fs,
  upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1578/fs,
  workdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1578/work
)
```

Meaning:

```text
lowerdir
→ read-only image snapshot/layers

upperdir
→ writable container layer

workdir
→ OverlayFS internal work area

rootfs
→ merged container filesystem
```

Conceptually:

```text
lowerdir(s)
     +
upperdir
     ↓
OverlayFS merged view
     ↓
container /
```

---

# 14. Mounts Sit on Top of OverlayFS

The final container filesystem is:

```text
OverlayFS merged root
+
extra mounts
```

Examples:

```text
ConfigMap
Secret
emptyDir
PVC
bind mounts
/etc/resolv.conf
/etc/hosts
/etc/hostname
```

If a mount is placed over an existing path, the original underlying content is hidden, not deleted.

---

# 15. lowerdir and File Deletion

Image lower layers are read-only from inside the container.

If a container runs:

```bash
rm /some/file
```

and the file exists only in a lower layer, OverlayFS does not remove the lower-layer file.

Instead it creates a whiteout in the upper layer.

Conceptually:

```text
lowerdir
└── file.txt

upperdir
└── whiteout for file.txt

merged view
└── file.txt appears deleted
```

This preserves shared image layers.

---

# 16. Why Shared Image Layers Stay Read-Only

Example:

```text
Image A
├── layer 1
└── layer 2

Image B
├── layer 1
└── layer 3
```

`layer 1` is shared.

If one container could directly modify or delete it, other containers using the same image layer could break.

That is why changes are written into the writable layer instead.

---

# 17. Do Not Manually Modify Runtime Storage

Avoid manually deleting data under:

```text
/var/lib/docker
/var/lib/containerd
/var/lib/kubelet
```

These locations are managed by Docker, containerd, and kubelet.

Manual changes can corrupt:

```text
images
snapshots
containers
volumes
runtime metadata
kubelet state
```

---

# 18. Docker Cleanup

Basic cleanup:

```bash
docker system prune
```

More aggressive:

```bash
docker system prune -a
```

Include unused volumes:

```bash
docker system prune --volumes
```

Docker only removes layers when no remaining object references them.

Mental model:

```text
layer still referenced
→ keep

no references remain
→ eligible for cleanup
```

---

# 19. Kubernetes Image Cleanup

There is no normal:

```text
kubectl system prune
```

Image cleanup happens on the node/runtime layer.

On real Kubernetes node:

```bash
sudo crictl images
sudo crictl ps
sudo crictl inspect <container-id>
sudo crictl rmi <image-id>
```

In kind:

```bash
docker exec devops-lab-worker crictl images
```

because the node itself is a Docker container.

Kubelet can also perform image garbage collection automatically when disk usage becomes high.

---

# 20. Docker CLI vs CRI

Docker path:

```text
docker CLI
   ↓
dockerd
   ↓
containerd
```

Kubernetes node path:

```text
kubelet / crictl
   ↓
CRI
   ↓
containerd
```

Equivalent ideas:

```text
docker ps       ↔ crictl ps
docker inspect  ↔ crictl inspect
docker images   ↔ crictl images
docker stop     ↔ crictl stop
```

They are different APIs, but can use the same underlying runtime.

---

# 21. Docker and containerd Sockets

Docker CLI commonly talks to:

```text
/var/run/docker.sock
```

dockerd commonly talks to containerd through:

```text
/run/containerd/containerd.sock
```

Conceptually:

```text
docker CLI
   ↓ HTTP over Unix socket
dockerd
   ↓ gRPC over Unix socket
containerd
```

The socket is the IPC transport.

The daemon processes handle concurrency.

---

# 22. containerd Is Also a Daemon

Runtime stack:

```text
docker CLI
   ↓
dockerd
   ↓
containerd
   ↓
containerd-shim
   ↓
container process
```

Roles:

```text
dockerd
→ Docker API and Docker-specific management

containerd
→ runtime lifecycle management

containerd-shim
→ per-container/task lifecycle intermediary

runc
→ low-level create/exec setup

kernel
→ processes, namespaces, mounts, signals, cgroups
```

---

# 23. How containerd Finds the Correct Shim

containerd tracks runtime state.

Mental model:

```text
container ID
   ↓
task
   ↓
shim
   ↓
process
```

For example:

```text
stop container B
    ↓
containerd receives task/container ID
    ↓
finds task runtime metadata
    ↓
identifies correct shim
    ↓
shim interacts with process/kernel
```

---

# 24. Real Kubernetes Nodes

On a real Kubernetes worker:

```text
VM / physical node
  ↓
kubelet
  ↓
containerd
```

No outer Docker layer is needed.

Troubleshooting flow:

```bash
kubectl get pod -o wide
```

Find the node.

Then on that node:

```bash
sudo crictl ps
sudo crictl inspect <container-id>
sudo journalctl -u kubelet
```

Storage inspection:

```bash
sudo findmnt -T /var/lib/containerd
sudo findmnt -T /var/lib/kubelet
```

---

# 25. Why Node Storage Must Be Protected

Because container and Pod mounts ultimately map back to node storage, modifying backing storage can directly affect workloads.

Sensitive paths include:

```text
/var/lib/docker
/var/lib/containerd
/var/lib/kubelet
```

Access should follow least privilege.

Read-only mounts should be used where appropriate.

Docker example:

```bash
docker run \
  -v /host/config:/config:ro \
  ...
```

Kubernetes:

```yaml
volumeMounts:
  - name: config
    mountPath: /config
    readOnly: true
```

---

# 26. PersistentVolumeClaim

PVC means:

```text
PersistentVolumeClaim
```

A PVC is a request for persistent storage.

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

Meaning:

```text
"I need persistent storage with these requirements."
```

---

# 27. PersistentVolume

PV means:

```text
PersistentVolume
```

A PV is the Kubernetes resource representing persistent storage.

Backing storage could be:

```text
AWS EBS
NFS
Ceph
SAN
local disk
other CSI-supported storage
```

Mental model:

```text
PV
→ Kubernetes representation of real persistent storage
```

---

# 28. StorageClass

StorageClass defines how storage should be provisioned.

Example:

```text
StorageClass: gp3
→ AWS EBS CSI driver
→ provision gp3 storage
```

PVC example:

```yaml
spec:
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

Meaning:

```text
"I need 20 GiB using the gp3 storage provisioning policy."
```

---

# 29. CSI

CSI stands for:

```text
Container Storage Interface
```

It is the standard interface between Kubernetes and storage providers.

Conceptually:

```text
PVC
   ↓
StorageClass
   ↓
CSI driver
   ↓
storage provider
   ↓
actual storage
   ↓
PV
```

---

# 30. PVC / PV / StorageClass Mental Model

Simple memory rule:

```text
PVC
= request

PV
= storage resource

StorageClass
= provisioning policy / storage type

CSI
= driver talking to storage provider

volumeMount
= where storage appears inside container
```

Full flow:

```text
Pod
 ↓
PVC
"I need 20 GiB"
 ↓
StorageClass
"Use this storage type"
 ↓
CSI driver
 ↓
storage backend
 ↓
PV
 ↓
PVC binds to PV
 ↓
volume mounted into Pod
```

---

# 31. Kubernetes Resource Terminology

Objects defined using:

```yaml
apiVersion:
kind:
metadata:
spec:
```

are Kubernetes API resources/objects.

Examples:

```text
Pod
Service
ConfigMap
Secret
PersistentVolume
PersistentVolumeClaim
Deployment
StatefulSet
DaemonSet
NetworkPolicy
```

Precise terminology:

```text
kind
→ resource type

metadata.name
→ name of a specific object

spec
→ desired configuration/state
```

So this wording is valid:

```text
PVC resource
→ creates a PersistentVolumeClaim object
→ requests persistent storage
```

---

# 32. Final Storage Mental Model

Everything eventually maps to Linux primitives.

```text
Kubernetes abstraction
        ↓
kubelet
        ↓
container runtime
        ↓
mount namespace
        ↓
filesystem mount
        ↓
node/host storage
        ↓
filesystem
        ↓
block device
```

In kind:

```text
Pod /config
   ↓
kubelet volume
   ↓
kind node /var/lib/kubelet
   ↓
Docker volume mounted at kind /var
   ↓
WSL /var/lib/docker/volumes/.../_data
   ↓
ext4
   ↓
/dev/sdd
```

Core takeaway:

```text
nothing is magic
```

Containers and Kubernetes mainly provide structured ways to control:

```text
which storage is visible
where it is mounted
who can access it
how long it lives
who manages it
```

using:

```text
mount namespaces
OverlayFS
volumes
permissions
container runtimes
kubelet
Kubernetes API abstractions
```