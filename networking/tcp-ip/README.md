# TCP/IP Practical Notes

## TCP Three-Way Handshake

```text
Client                 Server

SYN -------------------->

     <------------- SYN-ACK

ACK -------------------->

Connection established
```

## TCP provides:
- reliable delivery
- ordered delivery
- acknowledgements
- retransmission
- flow control
- congestion control

## Connection Errors

### Connection refused

**Usually means:**
- destination host is reachable
- TCP connection was actively rejected
- nothing is listening on that port
- service may be bound to the wrong interface

### Timeout

**Usually means:**
- no useful response was received
- packet may dropped
- firewall, routing, ACL or network path may be the problem

### HTTP error

**If you receive**
```
404
500
403
```
- then TCP connectivity already worked and the problem is at the application layer.

## Useful Commands
| Command      | Purpose                          |
| ------------ | -------------------------------- |
| `ip addr`    | Show interfaces and IP addresses |
| `ip route`   | Show routing table               |
| `ip neigh`   | Show ARP/neighbor table          |
| `ss -lntp`   | Show listening TCP sockets       |
| `curl`       | Test HTTP/HTTPS connectivity     |
| `ping`       | Test ICMP reachability           |
| `traceroute` | Inspect network path             |
| `dig`        | Test DNS resolution              |
| `nc`         | Test raw TCP/UDP connectivity    |

