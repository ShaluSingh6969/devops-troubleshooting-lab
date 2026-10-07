# Kubernetes Ingress Networking Lab

## Objective

Understand how Kubernetes Ingress works on top of Services and how external HTTP traffic reaches backend Pods.

This lab covers:

- Ingress vs Ingress Controller
- IngressClass
- ingress-nginx
- LoadBalancer Service in front of the controller
- NodePort access in kind
- path-based routing
- host-based routing
- rewrite rules
- Service backends
- common Ingress failure modes
- 404 vs 503 behavior

---

# 1. Starting State

Initially the cluster had no Ingress resources or controller.

Commands:

```bash
kubectl get pods -A | grep -i ingress
```

```bash
kubectl get ingressclass
```

```bash
kubectl get ingress -A
```

Observed:

```text
No Ingress Controller
No IngressClass
No Ingress resources
```

---

# 2. Ingress vs Ingress Controller

An Ingress resource is only configuration.

Example:

```text
/app-a → app-a-service
/app-b → app-b-service
```

The Ingress resource itself does not receive network traffic.

The actual reverse proxy is the Ingress Controller.

Mental model:

```text
Ingress
= routing rules

Ingress Controller
= actual reverse proxy implementing those rules
```

---

# 3. Installing ingress-nginx

The ingress-nginx controller was installed for the kind cluster.

After installation:

```bash
kubectl get pods -n ingress-nginx
```

showed:

```text
ingress-nginx-controller-...
1/1 Running
```

The controller Pod is the process actually receiving HTTP/HTTPS traffic.

---

# 4. IngressClass

Command:

```bash
kubectl get ingressclass
```

Observed:

```text
NAME    CONTROLLER
nginx   k8s.io/ingress-nginx
```

An Ingress can reference this using:

```yaml
spec:
  ingressClassName: nginx
```

This tells Kubernetes which controller should process the Ingress resource.

---

# 5. Controller Resources

Command:

```bash
kubectl get all -n ingress-nginx
```

Important resources included:

```text
Deployment
→ ingress-nginx-controller

Pod
→ actual NGINX controller

Service
→ ingress-nginx-controller

IngressClass
→ nginx
```

---

# 6. Service in Front of the Ingress Controller

The controller itself is a Pod, so it still needs a Kubernetes Service to expose it.

Command:

```bash
kubectl get svc -n ingress-nginx -o wide
```

Observed:

```text
ingress-nginx-controller
Type: LoadBalancer
ClusterIP: 10.96.181.67

HTTP:
80 → NodePort 32685

HTTPS:
443 → NodePort 31195
```

Important:

```text
The Ingress Controller Pod is NOT type LoadBalancer.

The Service in front of the controller is type LoadBalancer.
```

---

# 7. Why a Service Is Needed Before the Ingress Controller

The Ingress Controller is just another Pod.

Without a Service, it would only have an internal Pod IP.

The Service provides a stable entry point:

```text
external client
   ↓
Ingress Controller Service
   ↓
Ingress Controller Pod
```

---

# 8. kind and LoadBalancer Services

In a real cloud environment:

```text
LoadBalancer Service
   ↓
cloud provider provisions external load balancer
```

In this local kind cluster there is no cloud load balancer.

Therefore:

```text
EXTERNAL-IP = <pending>
```

But the Service still has NodePorts.

So the practical entry point in the lab was:

```text
NodeIP:32685
```

for HTTP.

---

# 9. Controller Reachability Test

Node IPs:

```text
worker1       172.21.0.2
control-plane 172.21.0.3
worker2       172.21.0.4
```

Tests:

```bash
curl -v http://172.21.0.2:32685
```

```bash
curl -v http://172.21.0.3:32685
```

```bash
curl -v http://172.21.0.4:32685
```

All returned:

```text
HTTP/1.1 404 Not Found
Server: nginx
```

This proved:

```text
NodePort
→ controller Service
→ Ingress Controller Pod
```

was working.

The 404 happened because no Ingress rule matched yet.

---

# 10. Backend Applications

Two Nginx Deployments were created:

```bash
kubectl create deployment app-a \
  --image=nginx:alpine
```

```bash
kubectl create deployment app-b \
  --image=nginx:alpine
```

Each Deployment was exposed through its own ClusterIP Service.

```bash
kubectl expose deployment app-a \
  --name=app-a-service \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

```bash
kubectl expose deployment app-b \
  --name=app-b-service \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

So:

```text
app-a-service
→ app-a Pod

app-b-service
→ app-b Pod
```

---

# 11. Ingress Rules

The Ingress resource used:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: apps-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /

spec:
  ingressClassName: nginx

  rules:
    - http:
        paths:

          - path: /app-a
            pathType: Prefix
            backend:
              service:
                name: app-a-service
                port:
                  number: 80

          - path: /app-b
            pathType: Prefix
            backend:
              service:
                name: app-b-service
                port:
                  number: 80
```

Applied using:

```bash
kubectl apply -f ingress.yaml
```

---

# 12. Ingress Backend Structure

In `networking.k8s.io/v1`, the correct structure is:

```yaml
backend:
  service:
    name: app-a-service
    port:
      number: 80
```

Incorrect nesting such as:

```yaml
backend:
  service:
    name: app-a-service
  port:
    number: 80
```

causes errors like:

```text
unknown field "spec.rules[0].http.paths[0].backend.port"
```

---

# 13. Inspecting the Ingress

Command:

```bash
kubectl describe ingress apps-ingress
```

Observed routing:

```text
Host   Path      Backends

*      /app-a    app-a-service:80
       /app-b    app-b-service:80
```

The controller also displayed the backend Pod endpoint behind each Service.

---

# 14. Meaning of Host `*`

The Ingress did not define a specific hostname.

Therefore:

```text
Host: *
```

means the rules can match requests regardless of the Host header.

Example:

```text
Host: 172.21.0.2:32685
Path: /app-a
```

still matches.

The backend is selected using the path.

---

# 15. Path-Based Routing

With the current configuration:

```text
any host + /app-a
→ app-a-service

any host + /app-b
→ app-b-service
```

Example:

```text
http://172.21.0.2:32685/app-a
→ app-a-service
```

```text
http://172.21.0.2:32685/app-b
→ app-b-service
```

---

# 16. Host-Based Routing

Host-based routing uses the HTTP Host header to select a backend.

Example:

```text
app-a.example.com
→ app-a-service

app-b.example.com
→ app-b-service
```

while both could use:

```text
/
```

Example configuration:

```yaml
rules:
  - host: app-a.example.com
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: app-a-service
              port:
                number: 80
```

The request still always contains both:

```text
Host
Path
```

The difference is which one is primarily used to distinguish applications.

---

# 17. Path-Based vs Host-Based Routing

Path-based:

```text
Host: example.com
/app-a → app-a-service
/app-b → app-b-service
```

Host-based:

```text
Host: app-a.example.com
/ → app-a-service

Host: app-b.example.com
/ → app-b-service
```

---

# 18. Initial 404 from Backend

After creating the Ingress, requests to:

```text
/app-a
/app-b
```

initially returned 404.

The Ingress routing itself worked.

The issue was:

```text
Ingress forwarded /app-a
→ backend Nginx received GET /app-a
```

but the default Nginx image only had content at:

```text
/
```

So the backend returned 404.

---

# 19. Path Rewrite

This annotation was added:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

Now:

```text
/app-a
→ rewritten to /
```

and:

```text
/app-b
→ rewritten to /
```

before being forwarded to backend Services.

After the rewrite:

```bash
curl http://172.21.0.2:32685/app-a
```

and:

```bash
curl http://172.21.0.2:32685/app-b
```

both returned:

```text
200 OK
Nginx welcome page
```

---

# 20. Full Ingress Traffic Path

The complete lab path is:

```text
WSL client
   ↓
NodeIP:32685
   ↓
NodePort
   ↓
ingress-nginx-controller Service
   ↓
Ingress Controller Pod
   ↓
Ingress rule
   ↓
path match
   ↓
rewrite
   ↓
app-a-service / app-b-service
   ↓
backend endpoint
   ↓
application Pod
```

---

# 21. Multiple Services Involved

There are two different types of Services in this architecture.

## Controller Service

```text
ingress-nginx-controller
```

Purpose:

```text
exposes the Ingress Controller
```

---

## Application Services

```text
app-a-service
app-b-service
```

Purpose:

```text
provide stable backends for application Pods
```

So:

```text
external client
   ↓
controller Service
   ↓
Ingress Controller
   ↓
application Service
   ↓
application Pod
```

---

# 22. Ingress Does Not Point Directly to Pods

The normal Kubernetes API model is:

```text
Ingress
→ Service
→ EndpointSlice
→ Pod
```

not:

```text
Ingress
→ Pod
```

This is important because Pod IPs are ephemeral.

Pods can:

```text
restart
be replaced
move nodes
scale up/down
```

Services provide the stable backend abstraction.

---

# 23. Controller May Proxy Directly to Pod Endpoints

Although the Ingress API references a Service, the controller can internally discover Service endpoints.

So implementation may effectively become:

```text
Ingress Controller
→ backend Pod IP
```

but configuration still uses:

```text
Ingress
→ Service
```

---

# 24. Meaning of `PORTS 80` in `kubectl get ingress`

Observed:

```text
NAME           CLASS   HOSTS   ADDRESS     PORTS
apps-ingress   nginx   *       localhost   80
```

The `80` means the Ingress currently exposes HTTP.

It does not mean:

```text
NodePort 80
```

and it is not specifically the backend Service port.

The actual lab path contained several port layers:

```text
NodePort:
32685

Ingress Controller Service:
80

Ingress listener:
80

Backend Service:
80

Backend Nginx:
80
```

---

# 25. HTTPS

The controller Service also exposed:

```text
443 → NodePort 31195
```

Once TLS is configured on an Ingress, it can typically show:

```text
80,443
```

Ingress TLS will be studied later.

---

# 26. LoadBalancer External IP

In kind:

```text
LoadBalancer Service
→ EXTERNAL-IP <pending>
```

because no external load-balancer implementation exists.

A local solution such as MetalLB could provide an external IP.

Mental model:

```text
external IP
   ↓
LoadBalancer Service
   ↓
Ingress Controller
```

Simply writing an arbitrary external IP into a Service does not automatically make the network advertise or route it.

---

# 27. Namespaces

Running:

```bash
kubectl get svc
```

only shows Services in the current namespace.

Application Services were in:

```text
default
```

while the controller Service was in:

```text
ingress-nginx
```

To view it:

```bash
kubectl get svc -n ingress-nginx
```

Or all Services:

```bash
kubectl get svc -A
```

---

# 28. Important Role Separation

```text
Ingress resource
→ HTTP routing configuration
```

```text
IngressClass
→ identifies which controller handles that configuration
```

```text
Ingress Controller
→ actual reverse proxy
```

```text
Controller Service
→ exposes the Ingress Controller
```

```text
Application Service
→ stable backend abstraction
```

```text
Pod
→ actual application
```

---

# 29. Final Mental Model

```text
External client
   ↓
Node / LoadBalancer entry
   ↓
Ingress Controller Service
   ↓
Ingress Controller
   ↓
Host + Path matching
   ↓
Ingress backend Service
   ↓
Endpoint
   ↓
Pod
```

The Ingress layer adds HTTP-aware routing on top of the Service networking already studied.

---

# 30. What to Remember

Do not memorize every generated resource name.

Remember:

```text
Ingress
= rules

Ingress Controller
= proxy

IngressClass
= controller selection

Controller Service
= exposes controller

Application Service
= backend abstraction

Pod
= actual application
```

And:

```text
Host + Path
→ determine route

Service
→ determines backend endpoints
```