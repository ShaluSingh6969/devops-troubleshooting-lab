# TLS Troubleshooting Incident Lab

## Objective

Build a practical troubleshooting approach for common HTTPS/TLS failures.

The focus is to identify exactly where a request fails in the dependency chain:

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

This is a troubleshooting dependency order, not an OSI layer ordering.

---

# Baseline: Healthy HTTPS Connection

Before investigating TLS failures, establish what a healthy request looks like.

Command:

```bash
curl -v https://example.com
```

Observed healthy flow:

```text
Hostname resolved
        ↓
TCP connection to :443 established
        ↓
TLS ClientHello
        ↓
TLS ServerHello
        ↓
Certificate received
        ↓
CertificateVerify
        ↓
TLS Finished
        ↓
Certificate verification OK
        ↓
ALPN selects HTTP/2
        ↓
HTTP GET
        ↓
HTTP 200
```

Important successful output from the lab:

```text
Connected to example.com (...) port 443

SSL connection using TLSv1.3

subjectAltName: host "example.com" matched cert's "example.com"

SSL certificate verify ok.

ALPN: server accepted h2

HTTP/2 200
```

This gives a known-good baseline for comparison with failure cases.

---

# Incident 1: Hostname Mismatch

## Scenario

The server presents a valid and trusted certificate, but the hostname expected by the client does not match the certificate SAN.

The lab deliberately tested this using:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -verify_hostname wrong.example.net
```

Observed error:

```text
verify error:num=62:hostname mismatch

Verification error: hostname mismatch

Verify return code: 62 (hostname mismatch)
```

## What Worked

```text
Network connectivity     ✓
TCP :443                 ✓
TLS negotiation          ✓
Certificate received     ✓
Certificate chain        ✓
Private-key proof        ✓
Hostname validation      ✗
```

## Cause

The client expected:

```text
wrong.example.net
```

but the certificate SAN contained:

```text
DNS:example.com
DNS:*.example.com
```

Therefore:

```text
wrong.example.net
    ≠
certificate SAN
```

## Troubleshooting Commands

Inspect SAN:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com </dev/null 2>/dev/null |
openssl x509 -noout -ext subjectAltName
```

Explicitly verify hostname:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -verify_hostname example.com
```

## Common Real-World Causes

- wrong certificate attached to a load balancer
- wrong Kubernetes TLS Secret
- wrong NGINX virtual host
- incorrect SNI
- DNS changed but certificate was not updated
- certificate does not contain the required SAN

## Key Lesson

```text
SNI
→ helps the server select a certificate

SAN
→ allows the client to verify that certificate
```

---

# Incident 2: Wrong SNI / Wrong Certificate Returned

## Scenario

The TCP connection is made to the expected destination, but the client deliberately sends another hostname through SNI.

Example:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername wrong.example.net \
  -showcerts
```

In the lab, the TLS endpoint returned a certificate associated with the hostname selected through SNI rather than simply using the hostname from `-connect`.

## Why This Happens

The TCP connection only establishes:

```text
destination IP:443
```

One IP may host multiple HTTPS services.

For example:

```text
203.0.113.10
    |
    +-- api.company.com
    +-- grafana.company.com
    +-- jenkins.company.com
```

The client sends:

```text
SNI = api.company.com
```

during the TLS ClientHello.

The server or load balancer uses that value to select:

```text
TLS virtual host
certificate
TLS configuration
```

## Troubleshooting Model

```text
DNS
 ↓
shared load-balancer IP
 ↓
TCP :443
 ↓
SNI
 ↓
certificate selection
 ↓
SAN verification
```

## Real-World Causes

- incorrect SNI from proxy
- incorrectly configured virtual host
- wrong ALB listener certificate
- default certificate being returned
- Kubernetes Ingress host/TLS mismatch
- reverse proxy serving fallback certificate

## Key Lesson

These are different values:

```text
TCP destination
SNI hostname
certificate SAN
HTTP Host / :authority
```

They normally align, but they are technically separate.

---

# Incident 3: Incomplete Certificate Chain

## Scenario

TCP connectivity works:

```bash
nc -vz api.company.com 443
```

but HTTPS fails with an error such as:

```text
SSL certificate problem:
unable to get local issuer certificate
```

## Interpretation

Because TCP succeeds:

```text
DNS       ✓
Routing   ✓
TCP :443  ✓
```

The failure happens during TLS certificate validation.

A common cause is a missing intermediate certificate.

Expected chain:

```text
Server certificate
       ↓
Intermediate CA
       ↓
Trusted Root CA
```

Broken chain:

```text
Server certificate
       ↓
   missing
       ↓
Trusted Root CA
```

## Troubleshooting Command

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com \
  -showcerts
```

Inspect:

```text
Certificate chain
Subject
Issuer
Verification
```

## Important Nuance

A missing intermediate is not the only possible cause.

The client may also:

- lack the required trusted root CA
- use an outdated CA trust store
- use a private/internal CA that has not been installed locally

Therefore inspect both:

```text
server-provided chain
+
client trust store
```

## Linux Trust Store

Common locations:

```text
/etc/ssl/certs/
/etc/ssl/certs/ca-certificates.crt
```

## Key Lesson

A certificate can be:

```text
valid
unexpired
correct hostname
```

and still fail because the client cannot construct a trusted CA path.

---

# Incident 4: Expired Certificate

## Scenario

The service is reachable but HTTPS fails with:

```text
certificate has expired
```

## Interpretation

The request already reached the TLS certificate-validation stage.

Therefore:

```text
DNS              ✓
Routing          ✓
TCP              ✓
TLS started      ✓
Certificate sent ✓
Validity check   ✗
```

## Inspect Certificate Dates

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com </dev/null 2>/dev/null |
openssl x509 -noout -subject -issuer -dates
```

Example:

```text
notBefore=...
notAfter=...
```

The current time must satisfy:

```text
notBefore
    ≤
current time
    ≤
notAfter
```

## Real-World Causes

- certificate renewal failed
- renewed certificate not deployed
- wrong certificate still attached
- load balancer still serving an old certificate
- Kubernetes Secret was not updated
- reverse proxy was not reloaded
- client/server system clock is badly incorrect

## TLS Termination Question

Always ask:

```text
Where does TLS terminate?
```

If architecture is:

```text
Client
 ↓ HTTPS
AWS ALB
 ↓ HTTP
Pod
```

then the ALB certificate must be investigated.

Replacing a certificate inside the Pod will not solve the problem.

## Key Lesson

Investigate the component that actually owns the TLS endpoint.

---

# Incident 5: HTTPS Client Connecting to Plain HTTP

## Scenario

A backend is listening with plain HTTP:

```text
127.0.0.1:8080
```

but the client tries:

```bash
curl https://127.0.0.1:8080
```

The TCP connection may succeed, but TLS fails.

A possible error is:

```text
wrong version number
```

## Why

Client sends:

```text
TLS ClientHello
```

but the server expects:

```text
plain HTTP
```

The protocols do not agree.

Conceptually:

```text
TCP                  ✓

Client expects TLS
       ↓
Server expects HTTP
       ↓
protocol mismatch
       ↓
TLS failure
```

## Common Real-World Causes

- load balancer backend protocol set to HTTPS instead of HTTP
- wrong backend port
- Ingress/reverse-proxy protocol misconfiguration
- application switched from HTTPS to HTTP
- service exposes a different protocol than expected

## Troubleshooting

First check raw TCP:

```bash
nc -vz host 8080
```

Then HTTP:

```bash
curl -v http://host:8080
```

Then HTTPS:

```bash
curl -vk https://host:8080
```

If HTTP works but HTTPS fails immediately, investigate protocol configuration.

---

# Incident 6: TLS Handshake Failure

## Scenario

The client reaches port `443`, but the TLS handshake fails before application traffic starts.

Possible error:

```text
tls alert handshake failure
```

## Possible Causes

- incompatible TLS versions
- no compatible cipher suites
- server expects a client certificate
- invalid server TLS configuration
- incorrect proxy/TLS configuration
- SNI-dependent configuration problem

## Troubleshooting Commands

Basic connection:

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com
```

Force TLS 1.3:

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com \
  -tls1_3
```

Force TLS 1.2:

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com \
  -tls1_2
```

This can help identify protocol compatibility problems.

---

# Incident 7: ALPN / Application Protocol Mismatch

## Scenario

TCP and TLS both succeed, but the expected application protocol is not negotiated correctly.

For example:

```text
Client supports:
h2
http/1.1

Server selects:
http/1.1
```

This might be acceptable for a normal web application.

But technologies such as gRPC commonly require HTTP/2.

## Test ALPN

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -alpn "h2,http/1.1"
```

Expected output may include:

```text
ALPN protocol: h2
```

## curl Example

```bash
curl -v https://example.com
```

Lab output showed:

```text
ALPN: curl offers h2,http/1.1

ALPN: server accepted h2

using HTTP/2
```

## Possible Causes

- proxy not configured for HTTP/2
- load balancer does not support expected protocol
- backend protocol configuration incorrect
- gRPC traffic forwarded as HTTP/1.1

---

# Incident 8: TLS Termination Failure

## Architecture

```text
Browser
 ↓ HTTPS
Load Balancer / Ingress
 ↓ HTTP
Backend
```

Users report:

```text
HTTPS broken
```

but:

```bash
curl http://backend:8080
```

works.

## Interpretation

The backend application is reachable.

The failure may exist at:

```text
TLS termination layer
```

Investigate:

```text
certificate
private key
SNI
TLS listener
certificate renewal
ALPN
listener rules
TLS versions
```

## AWS Example

```text
Client
 ↓
AWS ALB :443
 ↓
Target Group HTTP
 ↓
Application
```

Check:

```text
ALB HTTPS listener
ACM certificate
certificate expiry
SANs
listener certificate association
SNI selection
```

## Kubernetes Example

```text
Client
 ↓ HTTPS
Ingress Controller
 ↓ HTTP
Service
 ↓
Pod
```

Inspect:

```text
Ingress host rules
Ingress tls section
TLS Secret
tls.crt
tls.key
certificate SAN
Ingress controller logs
```

---

# TLS Troubleshooting Command Sequence

A practical investigation can follow this order.

## 1. Resolve Hostname

```bash
getent hosts api.company.com
```

or:

```bash
dig api.company.com
```

## 2. Check Routing

```bash
ip route get <destination-ip>
```

## 3. Check TCP Connectivity

```bash
nc -vz api.company.com 443
```

## 4. Inspect HTTPS

```bash
curl -v https://api.company.com
```

## 5. Inspect TLS Directly

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com
```

## 6. Inspect Certificate Chain

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com \
  -showcerts
```

## 7. Inspect Certificate Metadata

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com </dev/null 2>/dev/null |
openssl x509 -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
```

## 8. Test Hostname Verification

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com \
  -verify_hostname api.company.com
```

## 9. Test ALPN

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com \
  -alpn "h2,http/1.1"
```

---

# Troubleshooting Decision Tree

```text
Can hostname resolve?
 |
 +-- NO → DNS / NSS / resolver issue
 |
 YES
 |
Can TCP :443 connect?
 |
 +-- NO → routing / firewall / listener / network path
 |
 YES
 |
Does TLS handshake succeed?
 |
 +-- NO → TLS version / cipher / protocol / SNI / client-auth issue
 |
 YES
 |
Does certificate verify?
 |
 +-- NO → expiry / SAN / CA chain / trust store
 |
 YES
 |
Does expected ALPN protocol negotiate?
 |
 +-- NO → proxy / load balancer / protocol configuration
 |
 YES
 |
Does HTTP succeed?
 |
 +-- NO → HTTP routing / proxy / authentication / application
 |
 YES
 |
Application healthy
```

---

# Key Lessons

TLS problems should be investigated as specific failures rather than as generic "HTTPS issues."

A hostname mismatch means:

```text
certificate identity problem
```

An issuer-chain error means:

```text
certificate trust-path problem
```

An expired certificate means:

```text
certificate lifecycle problem
```

A `wrong version number` error often means:

```text
protocol mismatch
```

A TLS handshake failure may indicate:

```text
TLS compatibility or authentication configuration
```

A working backend with broken external HTTPS often points toward:

```text
TLS termination layer
```

The most important DevOps question during a TLS incident is often:

```text
Where does TLS terminate?
```

That tells you which component owns:

```text
certificate
private key
SNI handling
TLS configuration
certificate renewal
```

and therefore where troubleshooting should begin.