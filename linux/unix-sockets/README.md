# Linux Unix Sockets

## Goal

Understand how Unix domain sockets work and how Linux permissions control access to them.

## What is a Unix Domain Socket?

A Unix domain socket is a local IPC mechanism used for communication between processes on the same machine.

Unlike TCP sockets, it does not require:

- IP addresses
- TCP ports
- routing
- network interfaces

Example:

```text
client process
    |
    v
/run/docker.sock
    |
    v
server process
```

The kernel handles the communication locally.

## Docker Socket

Inspect the Docker socket:

```bash
ls -l /var/run/docker.sock
```

Observed:

```text
srw-rw---- root docker /var/run/docker.sock
```

The first character:

```text
s
```

means the file is a socket.

Permissions:

```text
root   -> read/write
docker -> read/write
others -> no access
```

Confirm type:

```bash
file /var/run/docker.sock
```

Output:

```text
socket
```

## Inspect Unix Socket Listeners

```bash
sudo ss -xlp | grep docker
```

Useful flags:

```text
-x = Unix sockets
-l = listening
-p = process information
```

Observed:

```text
/run/docker.sock
→ dockerd
```

`/var/run` normally points to `/run`, so these refer to the same socket:

```text
/var/run/docker.sock
/run/docker.sock
```

## Unix Socket vs TCP Socket

TCP:

```text
client
→ IP address
→ TCP port
→ server
```

Unix socket:

```text
client
→ filesystem socket path
→ server
```

Docker was not listening on a TCP port:

```bash
sudo ss -lntp | grep dockerd
```

No output was returned.

## Permission Troubleshooting

Current user:

```bash
id
```

showed:

```text
singh ∈ docker group
```

Confirmed with:

```bash
getent group docker
```

Therefore the user could access:

```text
/var/run/docker.sock
```

Test with an unprivileged user:

```bash
sudo -u nobody curl -v \
  --unix-socket /var/run/docker.sock \
  http://localhost/_ping
```

Result:

```text
Immediate connect fail for /var/run/docker.sock: Permission denied
```

This proved the failure was caused by Unix socket permissions, not Docker or networking.

## Troubleshooting Model

When Docker commands fail, check:

```text
1. Does docker.sock exist?
2. Is dockerd listening on it?
3. Does the current user have permission?
4. Is the user in the docker group?
5. Is the client using the correct socket path?
```

## Security Note

Access to the Docker socket is highly privileged.

Membership in the `docker` group is effectively root-equivalent in a normal rootful Docker setup because the user can control containers and host mounts.