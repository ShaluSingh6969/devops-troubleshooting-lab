# TCP Connection Troubleshooting Lab

## Objective

Understand how a successful TCP connection, a refused connection, and a timed-out connection appear from both:

- the client side
- packet captures using `tcpdump`

The lab also demonstrates how to distinguish application/listener problems from network/firewall problems.

---

## Environment

Tools used:

- Python HTTP server
- `curl`
- `nc` / Netcat
- `ss`
- `tcpdump`
- `iptables`

Test addresses and ports:

- Loopback IP: `127.0.0.1`
- HTTP server port: `8080`
- Unused test port: `9999`

---

# Scenario 1: Successful TCP Connection

## Start the HTTP Server

Create a small test file and start a Python HTTP server:

```bash
mkdir -p /tmp/network-lab
cd /tmp/network-lab
echo "tcpdump lab" > hello.txt

python3 -m http.server 8080 --bind 127.0.0.1
```

The server is listening on:

```text
127.0.0.1:8080
```

## Verify the Listening Socket

```bash
ss -lntp | grep ':8080'
```

`ss -lntp` shows listening TCP sockets and their associated processes.

## Start Packet Capture

```bash
sudo tcpdump -i lo -nn tcp port 8080
```

Options:

| Option | Meaning |
|---|---|
| `-i lo` | Capture traffic on the loopback interface |
| `-n` | Do not resolve IP addresses to hostnames |
| `-nn` | Do not resolve IP addresses or port numbers |
| `tcp port 8080` | Capture only TCP traffic involving port 8080 |

## Send the HTTP Request

From another terminal:

```bash
curl http://127.0.0.1:8080/hello.txt
```

Expected response:

```text
tcpdump lab
```

## Observed TCP Handshake

The capture showed a client ephemeral port connecting to server port `8080`.

Example from the lab:

```text
Client: 127.0.0.1:50538
Server: 127.0.0.1:8080
```

The first three packets were:

```text
50538 -> 8080    [S]
8080  -> 50538   [S.]
50538 -> 8080    [.]
```

This represents the TCP three-way handshake:

```text
Client                         Server

SYN ---------------------------->

    <---------------------- SYN-ACK

ACK ---------------------------->
```

TCP is now established.

## TCP Flags

| tcpdump Flag | Meaning |
|---|---|
| `[S]` | SYN |
| `[S.]` | SYN + ACK |
| `[.]` | ACK |
| `[P.]` | PSH + ACK |
| `[F.]` | FIN + ACK |
| `[R.]` | RST + ACK |

## HTTP Request and Response

After the TCP handshake, the client sent:

```text
Flags [P.]
GET /hello.txt HTTP/1.1
```

The server responded with:

```text
Flags [P.]
HTTP/1.0 200 OK
```

This demonstrates the layered flow:

```text
TCP connection established
        ↓
HTTP request
        ↓
HTTP response
```

HTTP communication occurs after TCP connectivity has been established.

## Sequence and Acknowledgement Numbers

An example packet from the capture was:

```text
seq 1:87, ack 1
```

TCP sequence numbers represent byte positions in the TCP data stream.

They are not simply packet numbers.

For example:

```text
seq 1:87
```

means that this TCP segment contains a range of bytes from the TCP stream.

## TCP Connection Teardown

The capture later showed:

```text
[F.]
```

This represents:

```text
FIN + ACK
```

FIN packets are used when one side wants to gracefully close the TCP connection.

Simplified complete connection:

```text
Client                         Server

SYN ---------------------------->
    <---------------------- SYN-ACK
ACK ---------------------------->

HTTP Request ------------------->
    <---------------- HTTP Response

    <---------------------- FIN
FIN ---------------------------->
    <---------------------- ACK
ACK ---------------------------->
```

---

# Scenario 2: TCP Connection Refused

For this test, TCP port `9999` was used with no application listening on it.

## Verify That Nothing Is Listening

```bash
ss -lntp | grep ':9999'
```

No output means there is no listening TCP socket on port `9999`.

## Start Packet Capture

```bash
sudo tcpdump -i lo -nn tcp port 9999
```

## Attempt the Connection

```bash
curl -v http://127.0.0.1:9999
```

The result was:

```text
Connection refused
```

The same TCP connectivity test can be performed with Netcat:

```bash
nc -vz 127.0.0.1 9999
```

Options:

| Option | Meaning |
|---|---|
| `nc` | Netcat |
| `-v` | Verbose output |
| `-z` | Test connectivity without sending application data |

## Packet Pattern

The packet capture showed approximately:

```text
Client                         Destination

SYN ---------------------------->

    <---------------------- RST
```

Or:

```text
[S]
[R.]
```

The client sends a SYN requesting a TCP connection.

Because nothing is listening on the destination port, the operating system responds immediately with a TCP reset.

## Interpretation

A connection refusal means that the destination was reachable enough to return an active negative response.

Common causes include:

- no application listening on the destination port
- wrong destination port
- application bound to another IP/interface
- firewall configured to actively reject the connection

Useful troubleshooting commands:

```bash
ss -lntp
```

```bash
nc -vz <host> <port>
```

```bash
sudo tcpdump -nn tcp port <port>
```

The first thing to investigate for a refusal is usually the listener.

---

# Scenario 3: TCP Connection Timeout

For this scenario, the HTTP server was still listening on port `8080`, but incoming TCP traffic was intentionally dropped.

## Verify That the Service Is Listening

```bash
ss -lntp | grep ':8080'
```

This proves that the application is running and listening before introducing the failure.

## Add a Temporary Firewall DROP Rule

```bash
sudo iptables -I INPUT 1 -i lo -p tcp --dport 8080 -j DROP
```

This rule silently drops incoming TCP packets destined for port `8080`.

Inspect the firewall rules:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

## Start Packet Capture

```bash
sudo tcpdump -i lo -nn tcp port 8080
```

## Attempt the Connection

```bash
curl -v --connect-timeout 5 http://127.0.0.1:8080/hello.txt
```

The same behavior can be tested with Netcat:

```bash
nc -vz -w 5 127.0.0.1 8080
```

Here:

```text
-w 5
```

means Netcat waits for a maximum of 5 seconds.

## Packet Pattern

The packet capture showed repeated SYN attempts:

```text
SYN

SYN retransmission

SYN retransmission

...
```

There was no:

```text
SYN-ACK
```

and no:

```text
RST
```

Simplified flow:

```text
Client                         Destination

SYN ----------------------------> DROP

SYN ----------------------------> DROP

SYN ----------------------------> DROP

          waiting...

          timeout
```

## Why Does TCP Retransmit?

The client receives no response.

It cannot immediately know whether:

- the packet was lost
- the destination host is unavailable
- routing is broken
- a firewall silently dropped the packet
- a cloud security rule blocked the connection
- there is another network-path problem

TCP therefore retransmits the SYN before eventually giving up.

## Remove the Firewall Rule

After completing the test:

```bash
sudo iptables -D INPUT -i lo -p tcp --dport 8080 -j DROP
```

Verify that the application becomes reachable again:

```bash
curl http://127.0.0.1:8080/hello.txt
```

---

# Successful vs Refused vs Timeout

| Situation | Typical Packet Pattern | Interpretation |
|---|---|---|
| Successful | `SYN → SYN-ACK → ACK` | TCP connection established |
| Connection refused | `SYN → RST` | Connection actively rejected |
| Connection timeout | `SYN → no response → retransmissions` | No useful response received |

---

# Troubleshooting Strategy

When an application cannot connect to another service, troubleshoot from the lowest failing layer upward.

```text
DNS
 ↓
Routing
 ↓
TCP
 ↓
TLS
 ↓
HTTP
 ↓
Application
```

## 1. DNS Resolution

Check whether the hostname resolves:

```bash
dig example.com
```

```bash
getent hosts example.com
```

`dig` tests DNS directly.

`getent` follows the Linux system name-resolution mechanism configured through `/etc/nsswitch.conf`.

For example:

```text
hosts: files dns
```

means Linux checks:

```text
/etc/hosts
```

before DNS.

The DNS resolver itself can be inspected with:

```bash
cat /etc/resolv.conf
```

A specific DNS server can also be queried directly:

```bash
dig @8.8.8.8 example.com
```

## 2. Routing

Check the routing table:

```bash
ip route
```

Determine which route Linux will use for a specific destination:

```bash
ip route get <destination-ip>
```

## 3. TCP Connectivity

Test whether a TCP connection can be established:

```bash
nc -vz <host> <port>
```

For example:

```bash
nc -vz example.com 443
```

## 4. Check the Listening Service

On the destination system:

```bash
ss -lntp
```

This helps identify:

- whether the application is running
- which port it is listening on
- which IP address/interface it is bound to

For example:

```text
127.0.0.1:8080
```

means loopback-only access.

```text
0.0.0.0:8080
```

means the service listens on all local IPv4 interfaces.

This does not automatically make the service publicly reachable because routing and firewall rules still apply.

## 5. HTTP/Application Layer

Test the HTTP connection:

```bash
curl -v http://<host>:<port>
```

For HTTPS:

```bash
curl -v https://<hostname>
```

If TCP connectivity succeeds but `curl` fails later, investigate higher layers such as:

- TLS
- HTTP
- authentication
- application behavior

## 6. Packet Capture

When the reason is still unclear:

```bash
sudo tcpdump -i <interface> -nn tcp port <port>
```

Packet capture helps determine whether:

- SYN leaves the client
- SYN-ACK returns
- RST is returned
- packets are retransmitted
- application data is exchanged

---

# Connection Refused vs Timeout

## Connection Refused

Typical pattern:

```text
SYN → RST
```

The destination actively responded.

First investigate:

- application process
- listening socket
- destination port
- bind address
- firewall REJECT rules

Example checks:

```bash
ss -lntp
```

```bash
nc -vz <host> <port>
```

```bash
tcpdump -nn tcp port <port>
```

## Connection Timeout

Typical pattern:

```text
SYN → no response
SYN retransmission
SYN retransmission
...
```

First investigate:

- routing
- firewall DROP rules
- security groups
- ACLs
- unreachable destination
- broken network path

A timeout does not automatically prove that a firewall is responsible.

---

# Key Takeaways

1. TCP establishes a connection using:

```text
SYN → SYN-ACK → ACK
```

2. Application protocols such as HTTP operate after the TCP connection has been established.

3. A connection refusal normally produces:

```text
SYN → RST
```

4. A timeout normally shows SYN retransmissions with no useful response.

5. `ss` helps inspect local listening sockets.

6. `nc` is useful for testing Layer 4 TCP connectivity without requiring HTTP.

7. `curl` tests higher-level application connectivity such as HTTP or HTTPS.

8. `tcpdump` shows what actually happened at the packet level.

9. Troubleshooting should follow the network stack rather than jumping randomly between tools:

```text
DNS
→ Routing
→ TCP
→ TLS
→ HTTP
→ Application
```

10. Error messages such as `Connection refused` and `Connection timed out` are useful evidence about where the failure may be occurring.
