# Kubernetes RBAC

RBAC = Role-Based Access Control.

It controls:

```text
Who
→ User / Group / ServiceAccount

Can do what
→ get / list / watch / create / update / delete / etc.

On which resources
→ pods / deployments / services / secrets / etc.

At what scope
→ namespace or cluster
```

## Core Objects

```text
Role
→ namespace-scoped permission definition

ClusterRole
→ cluster-scoped permission definition

RoleBinding
→ grants Role or ClusterRole permissions inside one namespace

ClusterRoleBinding
→ grants ClusterRole permissions cluster-wide
```

## ServiceAccount

ServiceAccounts are namespaced identities used by workloads.

Example identity:

```text
system:serviceaccount:default:pod-reader
```

A ServiceAccount can exist in one namespace but still be granted access to another namespace or cluster-wide through RBAC bindings.

Example:

```text
ServiceAccount:
default/pod-reader

RoleBinding in test-rbac
→ grants access inside test-rbac

ClusterRoleBinding
→ can grant access cluster-wide
```

---

## Role Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-role
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Core Kubernetes resources use:

```yaml
apiGroups: [""]
```

Examples:

```text
pods
services
configmaps
secrets
persistentvolumeclaims
```

Deployments belong to:

```text
apps/v1
```

so RBAC uses:

```yaml
apiGroups: ["apps"]
resources: ["deployments"]
```

---

## RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: pod-reader
    namespace: default
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```

This grants the Role permissions only inside the `default` namespace.

---

## ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

A ClusterRole does not automatically grant permissions.

It must be attached using a binding.

```text
ClusterRole + RoleBinding
→ permissions only in that RoleBinding's namespace

ClusterRole + ClusterRoleBinding
→ permissions cluster-wide
```

---

## Cross-Namespace Access

A ServiceAccount in one namespace can receive access to another namespace.

Example:

```yaml
subjects:
  - kind: ServiceAccount
    name: pod-reader
    namespace: default
```

inside a RoleBinding located in:

```text
test-rbac
```

means:

```text
default/pod-reader
→ receives permissions inside test-rbac
```

---

## ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-reader-cluster-binding
subjects:
  - kind: ServiceAccount
    name: pod-reader
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

This makes the ClusterRole permissions available cluster-wide.

---

## Subresources

Some Kubernetes operations use separate subresources.

Example:

```text
pods
pods/log
pods/exec
pods/portforward
deployments/scale
```

Reading Pods does not automatically mean reading logs.

Example:

```yaml
rules:
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
    verbs:
      - get
      - list
```

This allows Pod reading and log access.

---

## Checking Permissions

Check current identity permissions:

```bash
kubectl auth can-i --list
```

Check specific access:

```bash
kubectl auth can-i get pods
kubectl auth can-i delete pods
```

Check another identity:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:default:pod-reader
```

Check cluster-wide permissions:

```bash
kubectl auth can-i '*' '*' \
  --as=system:serviceaccount:default:pod-reader
```

---

## Useful Inspection Commands

```bash
kubectl get roles
kubectl get rolebindings

kubectl get clusterroles
kubectl get clusterrolebindings

kubectl describe role <name>
kubectl describe rolebinding <name>

kubectl api-resources
```

---

## Authentication vs Authorization

```text
Authentication
→ Who are you?

Authorization
→ What are you allowed to do?
```

Example with kubectl:

```text
kubectl
↓
kubeconfig
↓
client certificate / token / cloud identity
↓
API server authenticates identity
↓
RBAC checks RoleBindings / ClusterRoleBindings
↓
request allowed or denied
```

---

## Kubernetes Namespace vs Linux Namespace

They are unrelated concepts.

```text
Kubernetes namespace
→ logical grouping of API objects
→ RBAC / organization / naming scope

Linux namespace
→ kernel isolation mechanism
→ network / PID / mount / IPC / etc.
```

A Kubernetes namespace does not automatically provide network isolation.

Use resources such as:

```text
NetworkPolicy
```

for network-level restrictions.

---

## Mental Model

```text
Identity
→ User / Group / ServiceAccount

Permission definition
→ Role / ClusterRole

Binding
→ RoleBinding / ClusterRoleBinding

Scope
→ Namespace / Cluster

Goal
→ Least privilege
```

Always grant only the permissions a workload or user actually needs.