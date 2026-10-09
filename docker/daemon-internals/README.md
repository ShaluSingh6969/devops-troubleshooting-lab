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