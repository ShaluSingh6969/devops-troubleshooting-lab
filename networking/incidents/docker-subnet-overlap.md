# Incident — Docker Network Subnet Overlap

## Summary

A Docker bridge network can accidentally overlap with a VPN, corporate, cloud, or local subnet.

When this happens, Linux may choose the Docker bridge route instead of the correct external route because Docker's subnet is more specific than the default route.

This can cause failures such as:

```text
No route to host
timeout
host unreachable
service unreachable
```

---

## Scenario

Assume an internal corporate service uses:

```text
172.30.50.10
```

and should normally be reached through the default/VPN path.

Before the Docker conflict:

```bash
ip route get 172.30.50.10
```

returned:

```text
172.30.50.10 via 172.18.192.1 dev eth0
src 172.18.192.230
```

Expected path:

```text
172.30.50.10
   ↓
default route
   ↓
172.18.192.1
   ↓
eth0
   ↓
external/VPN network
```

---

## Creating the Conflict

A Docker network was created:

```bash
docker network create \
  --driver bridge \
  --subnet 172.30.0.0/16 \
  overlap-net
```

Docker created a Linux bridge similar to:

```text
br-e05e3a60235d
```

and added a route:

```text
172.30.0.0/16 dev br-e05e3a60235d
```

---

## Failure

After creating the Docker network:

```bash
ip route get 172.30.50.10
```

returned:

```text
172.30.50.10 dev br-e05e3a60235d
src 172.30.0.1
```

The intended destination was now being sent into the Docker bridge.

---

## Root Cause

Linux performs longest-prefix matching.

Available routes included:

```text
172.30.0.0/16 dev br-e05e3a60235d
default via 172.18.192.1 dev eth0
```

Destination:

```text
172.30.50.10
```

matches both:

```text
172.30.0.0/16
0.0.0.0/0
```

but `/16` is more specific than `/0`.

Therefore Linux selected:

```text
br-e05e3a60235d
```

instead of the default/VPN path.

---

## Why This Is Dangerous

Docker automatically creates host routing entries for bridge networks.

Therefore a Docker network can affect host traffic even when the destination has nothing to do with a running container.

The existence of the Docker route alone can hijack traffic.

---

## Diagnosis

First resolve the destination IP if necessary.

Then run:

```bash
ip route get <destination-ip>
```

Example:

```bash
ip route get 172.30.50.10
```

If the result unexpectedly shows:

```text
docker0
```

or:

```text
br-xxxxxxxx
```

investigate Docker networks.

Commands:

```bash
docker network ls
```

```bash
docker network inspect <network-name>
```

```bash
ip route
```

Compare Docker subnets against:

```text
VPN ranges
corporate subnets
local LAN
WSL subnet
cloud VPC/VNet networks
other Docker networks
```

---

## Correct Fix

Remove the overlapping Docker network:

```bash
docker network rm overlap-net
```

Verify:

```bash
ip route get 172.30.50.10
```

The route should return to:

```text
via 172.18.192.1 dev eth0
```

Then recreate the Docker network using a non-conflicting subnet:

```bash
docker network create \
  --driver bridge \
  --subnet 172.25.0.0/16 \
  overlap-net
```

Only use that subnet if it is known not to conflict with existing networks.

---

## Docker Compose Fix

Explicitly configure a safe subnet:

```yaml
networks:
  app-net:
    ipam:
      config:
        - subnet: 172.25.0.0/16
```

Then recreate the Compose network:

```bash
docker compose down
docker compose up -d
```

---

## Preventing Repeated Conflicts

If Docker repeatedly allocates problematic subnets, configure default Docker address pools.

Conceptual example:

```json
{
  "default-address-pools": [
    {
      "base": "172.25.0.0/16",
      "size": 24
    }
  ]
}
```

This makes Docker allocate smaller networks from a controlled address range.

---

## Temporary Workarounds

A host-specific route such as:

```bash
sudo ip route add 172.30.50.10/32 via <gateway>
```

can override a Docker `/16` because `/32` is more specific.

However, this is normally a workaround rather than a clean production fix.

It becomes fragile when many internal services or subnets are involved.

The preferred fix is to eliminate the overlapping Docker subnet.

---

## Troubleshooting Sequence

```text
Application cannot reach internal service
        ↓
Resolve hostname
        ↓
Get destination IP
        ↓
ip route get <destination-ip>
        ↓
Unexpected docker0 / br-*?
        ↓
docker network inspect
        ↓
Compare Docker subnet with VPN/corporate subnet
        ↓
Remove/recreate conflicting Docker network
        ↓
Verify route again
```

---

## Key Lesson

Docker networking participates in the host Linux routing table.

A Docker bridge is not isolated from host routing decisions.

When an internal service becomes unreachable only on one developer machine, Docker/VPN subnet overlap should be one of the first networking checks.