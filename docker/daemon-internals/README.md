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