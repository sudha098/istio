
# Istio mTLS & RBAC Policy Lab

This lab demonstrates how to configure **STRICT mutual TLS (mTLS)** and **fine-grained RBAC AuthorizationPolicies** in an Istio service mesh.
You will:

1. Enforce STRICT mTLS for workloads in a namespace
2. Validate that only mTLS traffic is accepted
3. Apply RBAC policies to **deny all traffic**
4. Add an **allow-list policy** permitting only specific HTTP methods between specific namespaces
5. Validate all expected behaviors using test pods

---

## 🧩 Prerequisites

* A Kubernetes cluster with Istio installed
* Sidecar injection enabled where required
* `kubectl` configured and functional

---

# Part 1 — Enforce STRICT mTLS in the product Namespace

### 1. Create PeerAuthentication (STRICT)

Apply:

```bash
kubectl apply -f peer_auth_identity.yaml
```

Example YAML (`peer_auth_identity.yaml`):

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: identity-peer-auth
  namespace: product
spec:
  mtls:
    mode: STRICT
```

### 2. Verify plaintext traffic is rejected

Run from **test namespace**, **without** sidecar:

```bash
kubectl exec -ti test -n test -- curl http://productpage.product.svc.cluster.local:9080
```

Expected:

```
curl: (56) Recv failure: Connection reset by peer
```

Because STRICT mTLS rejects plaintext.

### 3. Enable sidecar injection in test namespace

```bash
kubectl label namespace test istio-injection=enabled
kubectl delete pod test -n test
kubectl run test --image=nginx -n test
```

### 4. Test again — mTLS now succeeds

```bash
kubectl exec -ti test -n test -- curl http://productpage.product.svc.cluster.local:9080
```

You should now see the full HTML Bookstore application page.
This confirms STRICT mTLS is working and the caller now has a sidecar.

---

# Part 2 — Implement RBAC: DENY All + Selective Allow

You will now lock down all traffic to the **api** namespace and then allow only specific traffic from the **product** namespace.

---

## Step 1 — DENY all traffic to the api namespace

Apply:

```bash
kubectl apply -f deny_all.yaml
```

Example YAML (`deny_all.yaml`):

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: api
spec:
  {}
```

This creates a default-deny posture.

### Validate:

```bash
kubectl exec -ti lambda -n product -- curl http://httpbin.api.svc.cluster.local:8000
```

Expected:

```
RBAC: access denied
```

---

## Step 2 — Allow only GET and POST from product → api

Apply:

```bash
kubectl apply -f allow_from_product.yaml
```

Example YAML (`allow_from_product.yaml`):

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-from-product
  namespace: api
spec:
  rules:
  - from:
    - source:
        namespaces: ["product"]
    to:
    - operation:
        methods: ["GET", "POST"]
```

---

## Step 3 — Validate RBAC behavior

### ❌ Expect DENY for other methods (DELETE, PUT, PATCH, HEAD, OPTIONS)

Examples:

```bash
kubectl exec -ti lambda -n product -- curl -X DELETE http://httpbin.api.svc.cluster.local:8000/delete
# RBAC: access denied
```

### ✅ GET should succeed:

```bash
kubectl exec -ti lambda -n product -- curl http://httpbin.api.svc.cluster.local:8000/get
```

### ✅ POST should succeed:

```bash
kubectl exec -ti lambda -n product -- curl -X POST http://httpbin.api.svc.cluster.local:8000/post
```

If you curl `/`, the httpbin HTML home page loads (as seen in your output):

```bash
kubectl exec -ti lambda -n product -- curl http://httpbin.api.svc.cluster.local:8000
```

---

# Summary

| Component                                   | Purpose                                 | Result                                |
| ------------------------------------------- | --------------------------------------- | ------------------------------------- |
| **PeerAuthentication (STRICT)**             | Enforce mTLS in `product` namespace     | Plaintext rejected, sidecar → success |
| **AuthorizationPolicy: deny-all**           | Default deny posture in `api` namespace | All traffic blocked                   |
| **AuthorizationPolicy: allow-from-product** | Allow GET/POST from product → api       | GET/POST succeed                      |

You now have:

✔ STRICT mTLS
✔ Namespace-scoped RBAC
✔ Method-based filtering
✔ Cross-namespace traffic control


