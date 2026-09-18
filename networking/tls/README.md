# TLS Fundamentals and Troubleshooting

## Overview

TLS (Transport Layer Security) provides secure communication between applications.

For HTTPS over TCP, the practical connection flow is:

```text
Application wants https://example.com
        ↓
Name resolution
/etc/hosts → DNS
        ↓
Destination IP
        ↓
Routing decision
        ↓
Layer 2 next-hop delivery
        ↓
TCP connection to IP:443
        ↓
TLS handshake
        ↓
Encrypted HTTP
        ↓
Server application
```

TLS normally starts only after a TCP connection has been established.

For HTTPS:

```text
HTTP
 ↓
TLS
 ↓
TCP
 ↓
IP
```

A useful troubleshooting dependency order is:

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

This is a troubleshooting order, not an OSI layer ordering.

---

# What TLS Provides

TLS mainly provides:

- Confidentiality — traffic cannot normally be read by someone observing the network.
- Integrity — modifications to protected traffic can be detected.
- Authentication — the client can verify the identity of the server.

TCP alone does not provide these properties.

---

# TLS 1.3 Handshake

A simplified TLS 1.3 handshake looks like:

```text
Client                                      Server

TCP handshake
================================================

ClientHello ------------------------------->

                         <----------- ServerHello
                         <----------- EncryptedExtensions
                         <----------- Certificate
                         <----------- CertificateVerify
                         <----------- Finished

Finished ---------------------------------->

================================================
Encrypted application traffic

HTTP request ==============================>
                         <========= HTTP response
```

The TLS handshake performs three major tasks:

1. Negotiate TLS parameters.
2. Authenticate the server.
3. Establish shared session keys.

---

# ClientHello

The client starts TLS by sending a `ClientHello`.

It can contain information such as:

```text
Supported TLS versions
Supported cipher suites
Key exchange information
SNI hostname
ALPN protocols
```

Example from `curl -v`:

```text
TLSv1.3 (OUT), TLS handshake, Client hello (1)
```

---

# ServerHello

The server selects compatible TLS parameters and responds with a `ServerHello`.

Example:

```text
TLSv1.3 (IN), TLS handshake, Server hello (2)
```

---

# Certificate Authentication

The server sends its certificate during the TLS handshake.

The client checks things such as:

```text
Is the certificate valid for the requested hostname?
Is the certificate currently valid?
Is the certificate chain trusted?
Was the certificate correctly signed?
```

A TLS certificate contains or references information such as:

```text
Identity
Public key
Issuer
Validity period
Subject Alternative Names
Digital signature
```

---

# CertificateVerify

The server must prove that it owns the private key corresponding to the public key in its certificate.

The server signs TLS handshake information using its private key.

Conceptually:

```text
TLS handshake transcript
        ↓
Cryptographic hash
        ↓
Sign using server private key
        ↓
Digital signature
```

The client has:

```text
Handshake data
Server signature
Public key from certificate
```

It verifies:

```text
VERIFY(
    public_key,
    handshake_hash,
    signature
)
```

A successful verification proves that the server possesses the matching private key.

The private key itself is never transmitted.

---

# Certificate Signature vs CertificateVerify

These are two different signature checks.

```text
CA signature on certificate
        ↓
Proves certificate was issued by that CA

Server CertificateVerify signature
        ↓
Proves server possesses the certificate private key
```

Both are required for authenticated TLS.

---

# Public and Private Certificate Keys

The certificate contains or identifies the public key.

The server separately stores the private key.

Typical files might be:

```text
server.crt
server.key
```

The certificate can be distributed publicly.

The private key must remain secret.

Never commit private keys to Git.

---

# Key Exchange vs Session Keys

Certificate keys, key-exchange keys, and session keys perform different jobs.

## Certificate Key

Used primarily for server authentication.

```text
Certificate public/private key
        ↓
Prove server identity
```

## Ephemeral Key Exchange

In the observed TLS connection, X25519 was used:

```text
Server Temp Key: X25519
```

Both client and server create temporary key pairs:

```text
Client:
    private key
    public key

Server:
    private key
    public key
```

Only the public keys are exchanged.

```text
Client public key -------------------->
                     <---------------- Server public key
```

The client calculates:

```text
client private key
+
server public key
        ↓
shared secret
```

The server calculates:

```text
server private key
+
client public key
        ↓
same shared secret
```

The shared secret itself is never sent across the network.

An observer may see both public keys but cannot practically derive the shared secret without one of the private keys.

---

# Session / Traffic Keys

The shared secret is not normally used directly to encrypt HTTP traffic.

TLS uses a key derivation mechanism such as HKDF:

```text
X25519 key exchange
        ↓
Shared secret
        ↓
HKDF
        ↓
TLS traffic keys
```

Separate traffic keys can be derived for:

```text
Client → Server

Server → Client
```

These symmetric keys protect the actual application traffic.

Conceptually:

```text
Certificate key
= Who are you?

Exchange keys
= How do we create a shared secret?

Traffic/session keys
= What do we encrypt this connection with?
```

---

# Forward Secrecy

Ephemeral key exchange provides forward secrecy.

The certificate private key may remain in use for months, but ephemeral X25519 keys exist only temporarily.

Therefore, stealing the certificate private key later does not automatically allow an attacker to decrypt previously recorded sessions that used ephemeral key exchange.

---

# Man-in-the-Middle Protection

An attacker can observe the public X25519 keys.

However, simply replacing the exchanged keys would modify the authenticated TLS handshake.

The server's `CertificateVerify` signature authenticates the relevant handshake transcript.

Therefore:

```text
Passive attacker
→ sees public keys
→ cannot derive shared secret

Active attacker changes handshake/key exchange
→ authenticated transcript changes
→ signature verification fails
```

TLS therefore requires both:

```text
Key exchange
+
Authentication
```

---

# Certificate Chain

The observed certificate chain for `example.com` was:

```text
example.com
        ↓
Cloudflare TLS Issuing ECC CA 3
        ↓
SSL.com TLS Transit ECC CA R2
        ↓
SSL.com TLS ECC Root CA 2022
        ↓
Trusted CA path
```

With `openssl s_client`:

```text
depth=0 = leaf/server certificate
depth=1 = issuing intermediate
depth=2 = higher intermediate
depth=3 = higher CA/root path
```

Example:

```text
0 s:CN = example.com
  i:CN = Cloudflare TLS Issuing ECC CA 3
```

Here:

```text
s = Subject
i = Issuer
```

The issuer of one certificate should connect to the subject of the next certificate in the trust path.

---

# Root and Intermediate CAs

A common PKI hierarchy is:

```text
Root CA
   ↓
Intermediate CA
   ↓
Server certificate
```

Root CA private keys are highly sensitive and are therefore normally used less frequently.

Intermediate CAs issue server certificates.

The client generally already trusts root certificates through its local trust store.

---

# Linux CA Trust Store

`curl -v` showed:

```text
CAfile: /etc/ssl/certs/ca-certificates.crt
CApath: /etc/ssl/certs
```

These are used by the client to validate certificate chains.

A valid server certificate is not enough by itself.

The client must be able to build a trusted chain.

---

# Subject Alternative Name — SAN

Modern hostname verification primarily uses Subject Alternative Name.

The certificate inspected during the lab contained:

```text
X509v3 Subject Alternative Name:
    DNS:example.com
    DNS:*.example.com
```

Therefore it is valid for:

```text
example.com
api.example.com
www.example.com
```

The wildcard matches one DNS label.

For example:

```text
*.example.com
```

does not normally match:

```text
dev.api.example.com
```

---

# SNI vs SAN

SNI and SAN are related but perform different jobs.

## SNI

SNI is sent by the client in the TLS ClientHello.

It tells the TLS server:

```text
Which hostname am I requesting?
```

A load balancer serving multiple HTTPS applications can use SNI to select the appropriate certificate.

Example:

```text
Shared IP: 203.0.113.10

api.company.com
grafana.company.com
jenkins.company.com
```

The client may send:

```text
SNI = api.company.com
```

The TLS endpoint can then select the `api.company.com` certificate.

## SAN

SAN exists inside the returned certificate.

The client asks:

```text
Is this certificate valid for the hostname I expected?
```

Therefore:

```text
SNI
→ helps server SELECT a certificate

SAN
→ helps client VALIDATE the selected certificate
```

---

# TCP Destination, SNI and HTTP Host

These values are related but technically separate.

A connection can conceptually contain:

```text
TCP destination:
172.x.x.x:443

SNI:
api.company.com

HTTP Host / :authority:
api.company.com
```

Normally clients such as browsers and `curl` keep these aligned.

The sequence is:

```text
Hostname
 ↓
DNS
 ↓
IP:443
 ↓
TCP
 ↓
SNI
 ↓
TLS virtual host / certificate selection
 ↓
SAN validation
 ↓
HTTP Host or HTTP/2 :authority
 ↓
Application/backend routing
```

---

# Testing Hostname Verification

Correct SNI but intentionally incorrect verification name:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -verify_hostname wrong.example.net
```

Observed result:

```text
verify error:num=62:hostname mismatch
Verification error: hostname mismatch
Verify return code: 62 (hostname mismatch)
```

The TLS endpoint returned the correct `example.com` certificate because SNI was correct, but hostname verification failed because the expected hostname was intentionally incorrect.

For stricter OpenSSL behavior:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -verify_hostname wrong.example.net \
  -verify_return_error
```

---

# Testing SNI

A deliberately incorrect SNI can be supplied:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername wrong.example.net \
  -showcerts
```

In the lab, the TLS infrastructure returned a certificate associated with the SNI-selected hostname rather than the original hostname used by `-connect`.

This demonstrates:

```text
-connect
→ determines network destination

-servername
→ provides SNI to TLS server
```

---

# ALPN

ALPN stands for:

```text
Application-Layer Protocol Negotiation
```

It allows the client and server to choose which application protocol will run after TLS is established.

From `curl -v`:

```text
ALPN: curl offers h2,http/1.1
...
ALPN: server accepted h2
```

Therefore:

```text
Client offers:
HTTP/2
HTTP/1.1

Server chooses:
HTTP/2
```

A direct OpenSSL test:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -alpn "h2,http/1.1"
```

Without specifying `-alpn`, `openssl s_client` may report:

```text
No ALPN negotiated
```

ALPN is particularly important for:

```text
HTTP/2
gRPC
load balancers
reverse proxies
Kubernetes Ingress
```

---

# Observed TLS 1.3 Connection

`curl -v https://example.com` produced:

```text
SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519 / id-ecPublicKey
```

This can be separated into different cryptographic functions:

```text
TLSv1.3
→ protocol version

TLS_AES_256_GCM_SHA384
→ symmetric protection and key derivation/hash components

X25519
→ ephemeral key exchange

id-ecPublicKey
→ certificate public key type
```

A useful mental model:

```text
Authentication
→ certificate / ECDSA signatures

Key exchange
→ X25519

Shared secret
→ produced independently by both peers

Key derivation
→ HKDF

Application encryption
→ symmetric traffic keys / AES-GCM
```

---

# TLS Termination

TLS does not always terminate inside the application.

A very common architecture is:

```text
Client
   |
   | HTTPS
   v
Load Balancer
   |
   | HTTP
   v
Backend
```

The load balancer owns:

```text
Certificate
Private key
TLS configuration
```

This is called:

```text
TLS termination
```

Examples include:

```text
AWS ALB
NGINX
HAProxy
Kubernetes Ingress Controller
API Gateway
```

---

# TLS Termination with Re-encryption

Internal traffic can also be encrypted:

```text
Client
   |
   | HTTPS
   v
Load Balancer
   |
   | HTTPS
   v
Backend
```

These are two separate TLS connections:

```text
Client ↔ Load Balancer

Load Balancer ↔ Backend
```

They may use different certificates and different session keys.

---

# TLS Passthrough

Another architecture is:

```text
Client
   |
   | TLS encrypted
   v
Layer-4 Load Balancer
   |
   | same TLS connection
   v
Backend
```

The load balancer does not terminate TLS.

The backend owns the certificate/private key and performs the TLS handshake.

---

# Kubernetes TLS Example

A common Kubernetes architecture is:

```text
Browser
   ↓
HTTPS :443
   ↓
Ingress Controller
   ↓
HTTP
   ↓
Service
   ↓
Pod :8080
```

Ingress configuration might reference:

```yaml
spec:
  tls:
    - hosts:
        - api.company.com
      secretName: api-tls
```

The associated Secret may contain:

```text
tls.crt
tls.key
```

If TLS terminates at the Ingress, the Pod may serve only plain HTTP.

Therefore:

```text
External HTTPS broken
```

while:

```bash
curl http://pod-ip:8080
```

may still work.

---

# AWS ALB Example

Example architecture:

```text
Internet
   ↓
AWS ALB :443
   ↓
Target Group
   ↓
EC2 / ECS / EKS backend
```

The ALB can terminate TLS using an ACM certificate.

The backend target group can then use either:

```text
HTTP
```

or:

```text
HTTPS
```

depending on the architecture.

During a certificate-expiry incident, the first question should therefore be:

```text
Where does TLS terminate?
```

The component terminating TLS is where certificate, private-key, SNI and TLS configuration should normally be investigated first.

---

# Useful TLS Troubleshooting Commands

## Test TCP Port

```bash
nc -vz example.com 443
```

If this fails, TLS may never have started.

---

## Inspect the Full HTTPS Flow

```bash
curl -v https://example.com
```

This can expose:

```text
DNS resolution
TCP connection
TLS handshake
Certificate verification
ALPN
HTTP request
HTTP response
```

---

## Open a TLS Connection

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com
```

---

## Display Certificate Chain

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -showcerts </dev/null
```

---

## Inspect Subject, Issuer and Validity

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com </dev/null 2>/dev/null |
openssl x509 -noout -subject -issuer -dates
```

---

## Inspect SAN

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com </dev/null 2>/dev/null |
openssl x509 -noout -ext subjectAltName
```

---

## Explicitly Verify Hostname

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -verify_hostname example.com
```

---

## Test ALPN

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -alpn "h2,http/1.1"
```

---

# Common TLS Failure Patterns

| Error | Likely Problem |
|---|---|
| `Connection refused` | TCP listener, wrong port or active rejection |
| `Connection timed out` | Routing/firewall/security path |
| `certificate has expired` | Certificate validity |
| `hostname mismatch` | SAN / expected hostname / SNI configuration |
| `unable to get local issuer certificate` | Missing intermediate CA or client trust-store problem |
| `self-signed certificate` | Certificate signer is not trusted by client |
| `wrong version number` | TLS client communicating with plain HTTP or another protocol |
| `handshake failure` | TLS compatibility, cipher, TLS version, client-certificate or server configuration problem |
| HTTP `404`, `500`, etc. | TLS succeeded; investigate HTTP/application layer |

---

# Incomplete Certificate Chain

If TCP succeeds:

```bash
nc -vz api.company.com 443
```

but HTTPS returns:

```text
SSL certificate problem: unable to get local issuer certificate
```

then TCP is working.

Inspect:

```bash
openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com \
  -showcerts
```

A common problem is:

```text
Leaf certificate
        ↓
Missing intermediate
        ↓
Trusted root
```

However, the client trust store should also be checked because the expected CA may simply not be trusted locally.

---

# Systematic TLS Troubleshooting

When HTTPS fails, avoid jumping immediately to application logs.

Ask:

```text
1. Does the hostname resolve?

2. Does the OS have a route to the destination?

3. Can TCP connect to port 443?

4. Does the TLS handshake start?

5. Is the certificate trusted?

6. Does the SAN match the expected hostname?

7. Is the certificate within its validity period?

8. Is SNI selecting the correct TLS virtual host?

9. Does ALPN negotiate the expected protocol?

10. Does HTTP succeed?

11. Does the application behave correctly?
```

Useful progression:

```bash
getent hosts api.company.com

ip route get <destination-ip>

nc -vz api.company.com 443

curl -v https://api.company.com

openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com \
  -showcerts
```

---

# Key Takeaways

TLS should be understood as several related but distinct mechanisms:

```text
Certificate chain
      ↓
Establish trust

Certificate public/private key
      ↓
Authenticate server

Ephemeral key exchange
      ↓
Derive shared secret

HKDF
      ↓
Derive traffic keys

Symmetric encryption
      ↓
Protect application traffic
```

For DevOps troubleshooting, always determine:

```text
Where did the connection fail?

Where does TLS terminate?

Which component owns the certificate?

Which hostname was sent as SNI?

Which SANs exist in the certificate?

Can the client build a trusted CA chain?
```

Understanding those questions makes TLS errors much easier to isolate across reverse proxies, Kubernetes Ingress, AWS load balancers and backend applications.