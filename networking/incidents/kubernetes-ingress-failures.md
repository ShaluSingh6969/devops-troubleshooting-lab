# Kubernetes Ingress Failure Labs

## Purpose

Document common Ingress failure patterns and how to distinguish them.

---

# 1. Controller Reachable but No Ingress Rule

Request:

```bash
curl http://172.21.0.2:32685
```

Result:

```text
HTTP/1.1 404 Not Found
Server: nginx
```

Cause:

```text
request reached Ingress Controller
but
no Host/Path rule matched
```

Traffic path:

```text
client
   ↓
NodePort
   ↓
Ingress Controller
   ↓
no matching route
   ↓
404
```

Important:

```text
404 from ingress-nginx
does not necessarily mean network failure
```

It often means:

```text
controller reachable
routing rule did not match
```

---

# 2. Backend Path Does Not Exist

Ingress rule:

```text
/app-a
→ app-a-service
```

The backend default Nginx only served:

```text
/
```

Ingress forwarded:

```text
GET /app-a
```

to the backend.

Result:

```text
404
```

Cause:

```text
Ingress routing succeeded
but backend application path did not exist
```

Fix:

```yaml
nginx.ingress.kubernetes.io/rewrite-target: /
```

Then:

```text
/app-a
→ /
```

before reaching the backend.

---

# 3. Nonexistent Backend Service

The Ingress was changed to:

```yaml
backend:
  service:
    name: app-a-broken
    port:
      number: 80
```

but:

```text
app-a-broken
```

did not exist.

Controller logs showed:

```text
Error obtaining Endpoints for Service "default/app-a-broken":
no object matching key "default/app-a-broken" in local store
```

The request returned:

```text
HTTP/1.1 503 Service Temporarily Unavailable
```

---

# 4. Why 503 Occurred

The Ingress rule itself was valid.

The controller log showed:

```text
successfully validated configuration, accepting
```

So:

```text
Ingress syntax valid
```

but:

```text
backend Service unavailable
```

The path was:

```text
client
   ↓
Ingress Controller
   ↓
Ingress rule matches /app-a
   ↓
tries app-a-broken:80
   ↓
Service does not exist
   ↓
no usable upstream
   ↓
503
```

---

# 5. Healthy Route Continued Working

While `/app-a` returned 503, `/app-b` still returned:

```text
200 OK
```

Controller log:

```text
[default-app-b-service-80]
10.244.2.9:80
200
```

This proved:

```text
Ingress Controller healthy
NodePort path healthy
Ingress itself loaded
```

and isolated the issue specifically to:

```text
app-a backend configuration
```

---

# 6. Useful HTTP Status Interpretation

A useful first approximation:

```text
404
→ request reached controller
→ no route matched
OR backend application itself returned 404
```

```text
503
→ route matched
→ backend/upstream unavailable
```

This is not an absolute rule, so verify with controller logs and direct backend tests.

---

# 7. Distinguishing Controller 404 vs Backend 404

If the Ingress has no matching rule:

```text
Ingress Controller
→ generates 404
```

If the rule matches but the backend receives an invalid path:

```text
Ingress Controller
→ forwards request
→ backend application generates 404
```

To distinguish them:

```bash
kubectl logs -n ingress-nginx \
  deployment/ingress-nginx-controller
```

and test the backend Service directly:

```bash
kubectl exec client -- curl http://app-a-service/
```

```bash
kubectl exec client -- curl http://app-a-service/app-a
```

---

# 8. Troubleshooting Sequence

When an Ingress request fails:

```text
1. Can I reach the Ingress Controller?
```

Test:

```bash
curl -v http://<node-ip>:<controller-nodeport>
```

---

```text
2. Does the Host match?
```

Inspect:

```bash
kubectl describe ingress <name>
```

---

```text
3. Does the Path match?
```

Example:

```text
/app-a
```

---

```text
4. Does the referenced Service exist?
```

```bash
kubectl get svc
```

---

```text
5. Does the Service have endpoints?
```

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=<service>
```

---

```text
6. Is the configured Service port correct?
```

```bash
kubectl get svc <service> -o yaml
```

---

```text
7. Can the backend Service be reached directly?
```

```bash
kubectl exec <client-pod> -- \
  curl http://<service>
```

---

```text
8. Is the application path valid?
```

Test:

```bash
curl http://<service>/
```

versus:

```bash
curl http://<service>/<path>
```

---

```text
9. What does the Ingress Controller log say?
```

```bash
kubectl logs -n ingress-nginx \
  deployment/ingress-nginx-controller \
  --tail=100
```

---

# 9. Common Failure Map

```text
Cannot connect to controller
→ NodePort / LoadBalancer / controller Service / controller Pod problem
```

```text
404
→ Host or Path mismatch
→ or backend application returned 404
```

```text
503
→ Service missing
→ Service has no usable endpoints
→ backend unavailable
```

```text
One route works, another fails
→ controller itself likely healthy
→ inspect route-specific backend
```

---

# 10. Full Troubleshooting Path

```text
External client
   ↓
NodePort / LoadBalancer
   ↓
Controller Service
   ↓
Ingress Controller
   ↓
Host match
   ↓
Path match
   ↓
Backend Service exists?
   ↓
EndpointSlice healthy?
   ↓
Correct port?
   ↓
Backend Pod reachable?
   ↓
Application path valid?
```

---

# 11. Main Lesson

Do not treat:

```text
Ingress failed
```

as one network problem.

Break it into layers:

```text
external entry
controller reachability
Ingress routing
Service discovery
endpoint availability
backend networking
application behavior
```

The goal is to determine:

```text
Which layer produced the failure?
```