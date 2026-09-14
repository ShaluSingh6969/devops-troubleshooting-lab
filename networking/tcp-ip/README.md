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
| `nc -vz <host> <port>` | Test TCP connectivity to a host and port |


## TCP Troubleshooting with tcpdump

### Useful commands

| Command | Purpose |
|---------|---------|
| `tcpdump -i lo -nn tcp port 8080` | Capture TCP packets on loopback for port 8080 |
| `tcpdump -i eth0 -nn tcp port 443` | Capture TCP packets on `eth0` for HTTPS traffic |

### Important TCP Flags

| Flag | Meaning |
|---|---|
| `[S]` | SYN |
| `[S.]` | SYN + ACK |
| `[.]` | ACK |
| `[P.]` | PSH + ACK, often carrying application data |
| `[F.]` | FIN + ACK |
| `[R.]` | RST + ACK |

### Successful TCP Connection

A normal connection begins with the three-way handshake:

```text
Client                  Server

SYN --------------------> [S]
    <------------- SYN-ACK [S.]
ACK --------------------> [.]
```
After the handshake, application data can be exchanged. 

Example from an HTTP request

```
[P.] GET /hello.txt HTTP/1.1
[P.] HTTP/1.0 200 OK
```

### Connection Refused

Typical packet pattern:

Client                  Destination

SYN --------------------> [S]
    <---------------- RST  [R.]

A connection refusal normally means the destination responded immediately but the TCP connection was actively rejected.

Common causes:

- no application listening on the destination port
- application listening on another port
- application bound only to another interface/address
- firewall configured to actively reject traffic

#### Useful Checks:
```
ss -lntp
nc -vz <host> <port>
tcpdump -nn tcp port <port>
```

### Connection Timeout

SYN -------------------->
SYN -------------------->
SYN -------------------->
...
timeout

The client retransmits SYN packets because it receives no useful response.

**Possible causes:**

- firewall silently dropping packets
- cloud security group or ACL
- routing problem
- unreachable host
- broken network path

A timeout does not by itself prove that a firewall is responsible.

## Troubleshooting Mental Model

Troubleshoot in dependency order: name resolution → routing → transport connectivity → TLS → application protocol → application behavior.

DNS
 ↓
Routing
 ↓
TCP connection
 ↓
TLS
 ↓
HTTP
 ↓
Application

