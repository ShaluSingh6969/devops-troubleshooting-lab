# Docker Daemon Internals

## Goal

Understand the relationship between:

```text
docker CLI
dockerd
containerd
containerd-shim
runc
```

## Docker Runtime Architecture

Observed process:

```text
/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

This shows that `dockerd` communicates with `containerd` through:

```text
/run/containerd/containerd.sock
```

Architecture:

```text
docker CLI
   |
   v
/run/docker.sock
   |
   v
dockerd
   |
   v
/run/containerd/containerd.sock
   |
   v
containerd
   |
   v
containerd-shim-runc-v2
   |
   v
runc
   |
   v
container process
```

## Responsibilities

```text
dockerd
→ Docker API
→ Docker networking
→ volumes
→ image/build features
→ high-level Docker behavior

containerd
→ container lifecycle
→ runtime tasks
→ image/runtime management
→ snapshot management

containerd-shim
→ manages running container process lifecycle

runc
→ creates the container using Linux namespaces,
  cgroups and mounts
```

## Inspect Runtime Processes

```bash
ps -ef | grep -E 'dockerd|containerd' | grep -v grep
```

Observed:

```text
dockerd
containerd
containerd-shim-runc-v2
```

## Inspect Runtime Sockets

```bash
sudo ss -xlp | grep -E 'docker.sock|containerd.sock'
```

Observed:

```text
/run/docker.sock
→ dockerd

/run/containerd/containerd.sock
→ containerd
```

## Docker Engine API

Docker CLI is a client for the Docker Engine API.

Direct API test:

```bash
curl --unix-socket /var/run/docker.sock \
  http://localhost/_ping
```

Version API:

```bash
curl -s --unix-socket /var/run/docker.sock \
  http://localhost/version | python3 -m json.tool
```

Observed components included:

```text
Docker Engine
containerd
runc
docker-init
```

List containers directly through the API:

```bash
curl -s --unix-socket /var/run/docker.sock \
  http://localhost/containers/json
```

This is approximately what happens behind:

```bash
docker ps
```

Conceptually:

```text
docker ps
→ Docker Engine API
→ GET /containers/json
→ docker.sock
→ dockerd
```

## Why localhost Appears in curl

Example:

```bash
curl --unix-socket /var/run/docker.sock \
  http://localhost/_ping
```

The actual transport is the Unix socket:

```text
/var/run/docker.sock
```

It is not a TCP connection to localhost:80.

The HTTP URL is still required so curl can build the HTTP request.

## systemd Socket Activation

Observed:

```text
/run/docker.sock
→ dockerd
→ systemd
```

and Docker was started with:

```text
-H fd://
```

This means systemd can provide the listening socket to Docker.

Useful checks:

```bash
systemctl status docker
systemctl status docker.socket
```

## Docker vs Kubernetes Runtime Path

Docker:

```text
docker CLI
→ docker.sock
→ dockerd
→ containerd
→ runc
```

Modern Kubernetes:

```text
kubelet
→ CRI
→ containerd
→ runc
```

CRI means:

```text
Container Runtime Interface
```

Kubernetes no longer needs `dockerd` in the middle when using containerd directly.

## kind Observation

The system showed:

```text
namespace moby
```

for Docker-managed containers and:

```text
namespace k8s.io
```

for Kubernetes-managed containers inside kind nodes.

The environment therefore contained:

```text
WSL host
├── dockerd
│   └── host containerd
│       └── kind node containers
│
└── inside kind nodes
    ├── kubelet
    └── containerd
        └── Kubernetes workloads
```

## Key Takeaways

- Docker CLI communicates with `dockerd` through `docker.sock`.
- `dockerd` uses `containerd`.
- `containerd` manages runtime tasks.
- `runc` performs low-level container creation.
- Unix socket permissions control Docker access.
- Docker does not require a TCP port for local operation.
- Kubernetes can talk directly to containerd using CRI.

## Linux Namespaces and Container Isolation

### What is a container?

A container is not a lightweight virtual machine.

At the Linux level, a container is primarily a normal Linux process running with isolation provided by kernel features such as:

- Linux namespaces
- cgroups
- filesystem layers
- capabilities and other security controls

A useful mental model is:

```text
Normal Linux process
        +
PID namespace
Mount namespace
Network namespace
IPC namespace
UTS namespace
User namespace (optional)
        +
cgroups
        +
container filesystem
        ↓
Container
```

Containers share the host Linux kernel.

This is fundamentally different from virtual machines, where each VM normally has its own guest kernel.

```text
Virtual Machine

Application
    ↓
Guest operating system
    ↓
Guest kernel
    ↓
Hypervisor
    ↓
Host


Container

Application / Linux process
    ↓
Namespaces + cgroups
    ↓
Host Linux kernel
```

---

## What is a Linux Namespace?

A namespace is a Linux kernel feature that gives a process an isolated view of a particular system resource.

The underlying machine and kernel are still shared.

Namespaces change what a process can see.

A useful rule is:

```text
Namespaces → What can this process see?

Cgroups    → How much can this process use?
```

Docker combines multiple namespaces to create an isolated container environment.

---

## PID Namespace

A PID namespace isolates:

- process visibility
- process ID numbering
- process trees

During the lab, the kind control-plane container had a process that appeared differently depending on where it was observed.

From the host:

```text
systemd / container init → PID 882
```

Inside the container:

```text
systemd / container init → PID 1
```

These are not two different processes.

It is the same underlying Linux process being viewed from two different PID namespaces.

Conceptually:

```text
                    Linux Kernel
                         |
              same underlying process
                    /             \
                   /               \
       Host PID namespace      Container PID namespace
             PID 882                    PID 1
```

---

## Inspecting PID Namespaces

The host PID namespace was inspected with:

```bash
sudo readlink /proc/1/ns/pid
```

Example:

```text
pid:[4026532212]
```

The kind control-plane container's host PID was obtained using:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' devops-lab-control-plane)
```

Then its PID namespace was inspected:

```bash
sudo readlink /proc/$PID/ns/pid
```

Example:

```text
pid:[4026532297]
```

The two namespace identifiers were different:

```text
Host:
pid:[4026532212]

Container:
pid:[4026532297]
```

This proves that the host process and the container process belong to different PID namespaces.

The exact namespace ID is not important.

The important observation is:

```text
different namespace ID
        ↓
different PID namespace
        ↓
different view of processes
```

---

## Why does Docker use a separate PID namespace?

Without PID namespace isolation, a process inside a container could potentially see the host process tree, including processes such as:

```text
systemd
dockerd
containerd
sshd
other containers
user shells
```

A separate PID namespace gives the container its own process view.

For example:

```text
Container A
PID 1  nginx
PID 10 worker

Container B
PID 1  application
PID 15 worker

Host
PID 1    systemd
PID 325  dockerd
PID 882  Container A init
PID 950  Container B init
```

Each container can therefore have its own PID 1 even though all processes are ultimately running under the same host kernel.

Benefits include:

- process visibility isolation
- independent PID numbering
- cleaner process management
- isolation between containers
- container-specific signal handling

---

## Common Linux Namespaces Used by Containers

### PID namespace

Controls:

```text
process IDs
process visibility
process hierarchy
```

Example:

```text
Host sees process as PID 882
Container sees same process as PID 1
```

---

### Network namespace

Provides an isolated networking stack.

A network namespace can have its own:

```text
network interfaces
IP addresses
routing table
ARP / neighbor table
firewall rules
sockets
ports
```

This is why two containers can both listen on:

```text
0.0.0.0:80
```

without necessarily conflicting.

They are listening inside different network namespaces.

---

### Mount namespace

Controls which filesystem mount points a process sees.

This allows a container to have a filesystem view such as:

```text
/
├── bin
├── etc
├── usr
├── var
└── app
```

while the host has a completely different filesystem view.

The files still ultimately live on host storage, filesystem layers, volumes, or mounts.

---

### UTS Namespace

UTS stands for Unix Time-sharing System.

The UTS namespace primarily isolates:

```text
hostname
domain name
```

This allows a container to have a hostname such as:

```text
devops-lab-control-plane
```

without changing the hostname of the WSL host.

---

### IPC Namespace

IPC stands for Inter-Process Communication.

It isolates mechanisms such as:

```text
System V shared memory
POSIX message queues
semaphores
```

Processes in different IPC namespaces do not normally share these IPC resources.

---

### User Namespace

A user namespace can provide separate UID and GID mappings.

For example, a process may appear as:

```text
UID 0
```

inside a container while being mapped to a non-root UID on the host.

User namespaces are an additional security mechanism and are not necessarily enabled in every Docker configuration.

---

### Cgroup Namespace

Linux also provides a cgroup namespace.

It changes how processes see the cgroup hierarchy.

However, this should not be confused with the main purpose of cgroups themselves.

---

## Namespaces vs Cgroups

These two concepts solve different problems.

### Namespaces

Namespaces provide isolation.

They answer:

> What resources can this process see?

Examples:

```text
PID → which processes can I see?
NET → which network stack can I see?
MNT → which mount points can I see?
UTS → which hostname do I see?
IPC → which IPC resources can I see?
```

### Cgroups

Cgroups provide resource accounting and resource control.

They answer:

> How much of the machine can this process use?

Examples:

```text
CPU
memory
I/O
number of processes
```

Conceptually:

```text
Namespaces
    ↓
Isolation / visibility

Cgroups
    ↓
Resource management
```

Together they form much of the foundation of Linux containers.

---

## Important Mental Model

A container is not fundamentally a special process type.

Linux does not have a process type called:

```text
container process
```

Instead, container runtimes create ordinary processes and configure kernel features around them.

Conceptually:

```text
docker run nginx
        ↓
dockerd
        ↓
containerd
        ↓
containerd-shim
        ↓
runc
        ↓
Linux process
        +
namespaces
        +
cgroups
        +
filesystem isolation
```

After the process has been created, `runc` normally exits.

The actual application remains a normal process managed through the container runtime infrastructure.

---

## Key Takeaway

The simplest container mental model is:

```text
Container
=
Linux process
+
isolated namespaces
+
cgroup resource controls
+
container filesystem
```

Namespaces make the process believe it is operating inside its own environment while it still shares the host Linux kernel.

This is one of the major reasons containers are lighter than virtual machines.

## Docker Named Volumes, Mount Namespaces, and Container Filesystem Internals

### Goal

This lab focused on understanding how Docker storage and filesystem visibility work underneath the Docker abstraction.

Topics covered:

- Docker named volumes
- host-side Docker storage
- Linux mount namespaces
- `nsenter`
- `/proc/<PID>/mountinfo`
- Docker-managed files such as `/etc/resolv.conf`
- Docker logs and metadata
- OverlayFS layers
- how mounts hide underlying filesystem content
- difference between container metadata and live kernel state
- containerd / shim / runc process model

---

# 1. Creating a Docker Named Volume

Create a named volume:

```bash
docker volume create app-data
```

Inspect it:

```bash
docker volume inspect app-data
```

The volume exists independently of a specific container.

Example container:

```bash
docker run --rm -it \
  -v app-data:/data \
  alpine sh
```

Inside the container:

```sh
echo "hello from named volume" > /data/test.txt
cat /data/test.txt
```

After exiting the container, the container is deleted because of `--rm`, but the named volume remains.

A new container can access the same data:

```bash
docker run --rm \
  -v app-data:/data \
  alpine cat /data/test.txt
```

Expected output:

```text
hello from named volume
```

This proves:

```text
container deleted
       ↓
named volume survives
       ↓
new container can mount the same data
```

---

# 2. Where Docker Stores the Named Volume

The named volume was stored on the Docker host under:

```text
/var/lib/docker/volumes/app-data/_data
```

This is Docker-managed storage.

The container sees it as:

```text
/data
```

So conceptually:

```text
WSL host:

/var/lib/docker/volumes/app-data/_data
                ↓
            Linux mount
                ↓
Container:

/data
```

---

# 3. Named Volume vs Bind Mount

A bind mount uses a source path chosen by the user.

Example:

```bash
docker run \
  -v ~/docker-mount-lab/data:/app/data \
  alpine
```

Conceptually:

```text
Bind mount

User-managed host path
~/docker-mount-lab/data
        ↓
container /app/data
```

A named volume uses a Docker-managed source path.

```text
Named volume

Docker-managed host path
/var/lib/docker/volumes/app-data/_data
        ↓
container /data
```

At the Linux kernel level, both are ultimately mounts.

The main difference is who manages the source path and lifecycle.

```text
Bind mount
→ user manages source path

Named volume
→ Docker manages source path
```

---

# 4. Inspecting the Container Mount

Start a long-running container:

```bash
docker run -d \
  --name volume-test \
  -v app-data:/data \
  alpine sleep 10000
```

Get the container's host PID:

```bash
docker inspect \
  --format '{{.State.Pid}}' \
  volume-test
```

Example:

```text
73971
```

Inspect its mount table from the host:

```bash
sudo grep '/data' /proc/73971/mountinfo
```

Observed output:

```text
2076 2050 8:48 /var/lib/docker/volumes/app-data/_data /data rw,relatime master:301 - ext4 /dev/sdd rw,discard,errors=remount-ro,data=ordered
```

Important fields:

```text
2076
```

Mount ID.

```text
2050
```

Parent mount ID.

```text
8:48
```

Major:minor device number.

```text
/var/lib/docker/volumes/app-data/_data
```

Source path.

```text
/data
```

Mount point inside the container's mount namespace.

```text
rw,relatime
```

Mount options.

After the `-` separator:

```text
ext4 /dev/sdd
```

Underlying filesystem and block device.

---

# 5. Container Mount Namespace

A Docker container is not a virtual machine.

It is fundamentally a set of Linux processes running with:

```text
namespaces
+
cgroups
+
filesystem mounts
+
runtime metadata
```

The container process belongs to namespaces such as:

```text
mount namespace
PID namespace
network namespace
UTS namespace
IPC namespace
...
```

We can enter one of these namespaces using:

```bash
nsenter
```

---

# 6. Entering Only the Mount Namespace

We first tried:

```bash
sudo nsenter -t 73971 -m sh
```

This means:

```text
target PID: 73971
-m         → enter its mount namespace
```

The new shell therefore had:

```text
mount namespace → container
PID namespace   → host
```

This created a mixed namespace view.

Running:

```sh
mount | grep /data
```

failed with:

```text
mount: no /proc/mounts
```

The reason is that `/proc` is closely related to the PID namespace it represents.

We had:

```text
container mount namespace
+
host PID namespace
```

which made `/proc` inconsistent for tools expecting the namespaces to match.

---

# 7. Entering Mount + PID Namespace

The correct command was:

```bash
sudo nsenter \
  -t 73971 \
  -m \
  -p \
  --mount-proc \
  sh
```

Meaning:

```text
-m
→ enter container mount namespace

-p
→ enter container PID namespace

--mount-proc
→ create a /proc view matching that PID namespace
```

Now the environment became consistent:

```text
container mount namespace
+
container PID namespace
+
matching /proc
```

Commands such as:

```sh
mount
```

and:

```sh
cat /proc/self/mountinfo
```

worked properly.

Important lesson:

```text
Namespaces are independent dimensions.

Entering only one namespace can create a mixed environment.
```

---

# 8. docker exec Mental Model

When we run:

```bash
docker exec -it container sh
```

we are not literally "entering a container."

Docker creates a new process in the container's existing execution context.

Conceptually:

```text
docker CLI
   ↓
dockerd
   ↓
containerd
   ↓
containerd-shim / runtime
   ↓
new shell process
   ↓
container namespaces + cgroups
```

The shell then sees:

```text
container mount namespace
container PID namespace
container network namespace
container hostname
container cgroups
...
```

This creates the illusion of entering a box.

A better mental model is:

```text
docker exec
=
start another Linux process
inside the container's namespace/cgroup context
```

---

# 9. Docker Runtime Process Chain

The approximate runtime chain is:

```text
docker CLI
   ↓
dockerd
   ↓
containerd
   ↓
containerd-shim
   ↓
runc
   ↓
Linux container process
```

Roles:

```text
dockerd
→ Docker API and higher-level Docker management

containerd
→ container lifecycle management

containerd-shim
→ stays associated with the container process

runc
→ performs low-level process/namespace/cgroup setup

Linux kernel
→ actually manages processes, namespaces, signals, mounts, etc.
```

`runc` does not normally stay running for the lifetime of the container.

The shim remains around.

---

# 10. Signals and Container Lifecycle

For:

```bash
docker stop container
```

the conceptual flow is:

```text
docker CLI
   ↓
dockerd
   ↓
containerd
   ↓
containerd-shim
   ↓
Linux kernel
   ↓
container process receives signal
```

The shim is still a normal Linux process.

It interacts with the workload through normal kernel mechanisms.

---

# 11. Docker Metadata vs Kernel State

Docker commands do not always execute something inside the container.

Examples:

```bash
docker inspect
docker ps
docker logs
docker stop
```

These are mainly runtime/control-plane operations.

Docker combines information from:

```text
persistent host-side metadata
+
containerd/runtime state
+
live kernel state
```

---

# 12. Docker Root Directory

Command:

```bash
docker info | grep "Docker Root Dir"
```

Observed:

```text
Docker Root Dir: /var/lib/docker
```

This is where Docker stores much of its host-side state.

Examples include:

```text
/var/lib/docker/
├── containers/
├── volumes/
├── image-related data
├── networking metadata
└── other runtime metadata
```

---

# 13. Container Metadata Directory

For the test container:

```text
/var/lib/docker/containers/<container-id>/
```

contained files such as:

```text
config.v2.json
hostconfig.json
hostname
hosts
resolv.conf
mounts/
checkpoints/
<container-id>-json.log
```

Rough meaning:

```text
config.v2.json
→ container configuration/runtime metadata

hostconfig.json
→ host-side container configuration

hostname
→ hostname file exposed inside container

hosts
→ container /etc/hosts data

resolv.conf
→ container DNS configuration

-json.log
→ container stdout/stderr for json-file logging driver
```

---

# 14. Docker Logs

The container log path can be found with:

```bash
docker inspect \
  --format '{{.LogPath}}' \
  volume-test
```

Example:

```text
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

The test container was running:

```text
sleep 10000
```

which produces no stdout/stderr.

Therefore the log file size was:

```text
0 bytes
```

This is expected.

Conceptually:

```text
container process stdout/stderr
        ↓
runtime/shim
        ↓
Docker logging driver
        ↓
host-side log storage
```

Then:

```bash
docker logs container
```

reads from the configured logging backend.

---

# 15. Docker DNS Configuration

The container-specific DNS file existed on the host at:

```text
/var/lib/docker/containers/<container-id>/resolv.conf
```

Contents:

```text
nameserver 10.255.255.254
```

Docker reported:

```text
# Based on host file: '/etc/resolv.conf' (legacy)
```

Inside the container:

```bash
cat /etc/resolv.conf
```

showed the same content.

So the flow is:

```text
WSL host /etc/resolv.conf
        ↓
Docker reads host DNS configuration
        ↓
Docker generates container-specific resolv.conf
        ↓
Docker mounts it into container
        ↓
container sees /etc/resolv.conf
```

---

# 16. Proving resolv.conf Is a Mount

Inside the container namespace:

```bash
cat /proc/self/mountinfo | grep 'resolv'
```

Observed:

```text
2077 2050 8:48 /var/lib/docker/containers/<container-id>/resolv.conf /etc/resolv.conf rw,relatime - ext4 /dev/sdd ...
```

This proves:

```text
host-side file:

/var/lib/docker/containers/<container-id>/resolv.conf

        ↓

mounted into container at:

/etc/resolv.conf
```

Docker does similar things for:

```text
/etc/hosts
/etc/hostname
/etc/resolv.conf
```

---

# 17. OverlayFS Container Filesystem

The container root filesystem uses OverlayFS.

The important concepts are:

```text
lowerdir
→ read-only image layers

upperdir
→ container writable layer

workdir
→ internal OverlayFS working directory

merged view
→ combined filesystem presented to container
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

# 18. lowerdir

`lowerdir` contains read-only filesystem layers from the image.

Example:

```text
Alpine base image files
```

Multiple lower layers may exist.

Files in lower layers are normally not modified directly.

---

# 19. upperdir

`upperdir` is the container's writable layer.

When the container creates a new file:

```text
new file
→ upperdir
```

If the container modifies a file that originally exists in a lower layer, OverlayFS performs:

```text
copy-up
```

The file is copied into the upper layer and modified there.

The upper-layer version then takes precedence in the merged view.

---

# 20. merged View

The merged filesystem is what the container sees as:

```text
/
```

Conceptually:

```text
lower layers
   +
upper layer
   ↓
merged filesystem
```

The container does not normally see the lower and upper directories separately.

It sees one filesystem tree.

---

# 21. Mounts on Top of the OverlayFS Root

Docker can add extra mounts on top of paths inside the merged filesystem.

Examples:

```text
named volumes
bind mounts
/etc/resolv.conf
/etc/hosts
/etc/hostname
```

So the final filesystem view is more accurately:

```text
OverlayFS merged root
        +
additional mounts
        ↓
container filesystem view
```

---

# 22. Mounts Hide Underlying Content

Suppose the image already contains:

```text
/data/file.txt
```

Then Docker mounts a volume onto:

```text
/data
```

The original merged filesystem content is not deleted.

It is simply hidden by the mount.

Conceptually:

```text
OverlayFS merged view:

/data
├── file.txt
└── old-data.txt

        ↓ volume mounted over /data

Container now sees:

/data
└── volume contents
```

The original files still exist underneath the mount.

If the mount disappears, the underlying filesystem becomes visible again.

---

# 23. Example: resolv.conf

The OverlayFS root may already contain:

```text
/etc/resolv.conf
```

But Docker mounts:

```text
/var/lib/docker/containers/<id>/resolv.conf
```

onto:

```text
/etc/resolv.conf
```

Therefore:

```text
OverlayFS /etc/resolv.conf
        ↓ hidden by
Docker-managed file mount
        ↓
container sees Docker resolv.conf
```

---

# 24. Example: Named Volume

Likewise:

```text
OverlayFS /data
        ↓ hidden by
named volume mount
        ↓
/var/lib/docker/volumes/app-data/_data
```

The volume is separate from the container writable layer.

---

# 25. Final Container Filesystem Mental Model

The clean mental model is:

```text
container filesystem
=
OverlayFS merged root
+
Docker-managed mounts
+
user-configured mounts
```

More visually:

```text
Image layer(s)
   ↓
lowerdir
      \
       \
        + OverlayFS
       /
upperdir
   ↓
merged root
   ↓
container /
   +
   ├── named volume mounts
   ├── bind mounts
   ├── /etc/resolv.conf mount
   ├── /etc/hosts mount
   └── /etc/hostname mount
```

---

# 26. Important Takeaways

- A Docker named volume is still a Linux mount underneath.
- Docker manages the source path for named volumes.
- Bind mounts use user-selected host paths.
- Containers are normal Linux processes with namespaces and cgroups.
- `docker exec` creates a new process in the container's namespace/cgroup context.
- `nsenter` can manually enter selected namespaces.
- Entering only some namespaces can create inconsistent mixed views.
- `/proc` should match the PID namespace when inspecting processes.
- Docker stores persistent metadata under `/var/lib/docker`.
- `docker logs` reads from Docker's logging backend rather than entering the container.
- Docker creates container-specific files such as `resolv.conf`.
- These files are mounted into the container filesystem.
- OverlayFS provides the base merged root.
- Extra mounts can hide paths from the OverlayFS merged view.
- Hidden content is not deleted; it is simply covered by the mount.
- The final container filesystem is assembled from image layers, writable layers, and additional mounts.

---

# 27. Core Mental Model

```text
Container
≠ virtual machine

Container
=
Linux process(es)
+
namespaces
+
cgroups
+
OverlayFS
+
mounts
+
runtime metadata
```

And:

```text
Docker abstraction
        ↓
Linux kernel primitives
```

Understanding the Linux primitives underneath Docker makes container troubleshooting much easier.