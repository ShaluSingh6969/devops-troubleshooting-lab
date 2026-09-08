# Incident — Local HTTP Service Reachability

## Scenario

A Python HTTP server is running on a Windows laptop:

```cmd
python -m http.server 8080 --bind 0.0.0.0
```

## Observation

The following command failed:
```
curl http://192.XX.XX.XX
```
because curl defaulted to TCP port 80. The service was actually listening on port 8080.

## Root Cause

The destination IP was correct, but the destination port was wrong.

## Networking Principle

- **Layer 3** identifies the destination host.

- **Layer 4** identifies the destination service using a port.

## Additional Findings

- 127.0.0.1 means loopback/local-only access.
- 0.0.0.0 means listen on all local IPv4 interfaces.
- same-subnet clients can communicate directly without using the default gateway.
- different-subnet clients require routing.
- private IP addresses such as 192.168.x.x are not directly reachable from the public internet.
- internet reachability normally requires routing/NAT/firewall configuration.
Validation