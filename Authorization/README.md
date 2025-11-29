
# Istio Service Mesh Authorization Lab

This lab demonstrates how to configure **authentication and authorization** in an Istio Service Mesh. By the end of this lab, you will learn how to:

* Deploy sample applications in Istio-enabled namespaces
* Enforce **mutual TLS (mTLS)** across workloads
* Apply **AuthorizationPolicies** to control access to services
* Use **ALLOW** and **DENY** policies to secure endpoints

---

## Prerequisites

* Kubernetes cluster with Istio installed
* `kubectl` configured to access the cluster
* Istio injection enabled for the default namespace:

```bash
kubectl get ns --show-labels
```

Example output:

```
NAME              STATUS   AGE   LABELS
default           Active   50m   istio-injection=enabled
istio-system      Active   88s
...
```

---

## Step 1: Deploy HTTPBin Application

Deploy HTTPBin using Istio sample YAML:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml
```

Verify pod status:

```bash
kubectl get pods
```

Verify the HTTPBin service:

```bash
kubectl get svc
kubectl exec -it -n test test -- curl --head httpbin.default.svc.cluster.local:8000
```

---

## Step 2: Enable mTLS with PeerAuthentication

Create a global PeerAuthentication policy in the `istio-system` namespace to enforce **mutual TLS**:

```bash
kubectl apply -f peer_auth_global.yaml
```

Verify access from the `test` namespace pod:

```bash
kubectl exec -ti test -n test -- curl --head http://httpbin.default.svc:8000
```

Expected result: **HTTP/1.1 200 OK**

---

## Step 3: Create AuthorizationPolicy

Create an `ALLOW` policy to permit **GET requests from the test namespace**:

```bash
kubectl apply -f auth_policy.yaml
kubectl get authorizationpolicies
```

Test the policy:

```bash
# HEAD request should fail
kubectl exec -ti test -n test -- curl --head http://httpbin.default.svc:8000
# GET request should succeed
kubectl exec -ti test -n test -- curl http://httpbin.default.svc:8000
```

---

## Step 4: Update AuthorizationPolicy

1. Allow **HEAD requests** in addition to GET.
2. Allow access from the `app` namespace:

```bash
kubectl apply -f auth_policy.yaml
```

Verify access:

```bash
kubectl exec -ti test -n app -- curl --head http://httpbin.default.svc:8000
```

Expected result: **HTTP/1.1 200 OK**

---

## Step 5: Apply DENY Policy

Block access to `/delay` endpoints:

```bash
kubectl apply -f auth_policy_deny.yaml
kubectl get authorizationpolicies
```

Test access:

```bash
kubectl exec -ti test -n test -- curl http://httpbin.default.svc:8000/delay/1
# Expected: RBAC: access denied

kubectl exec -ti test -n test -- curl http://httpbin.default.svc:8000/get
# Expected: GET succeeds
```

---

## Step 6: Use Selectors for Specific Workloads

Update `auth_policy.yaml` to target only workloads labeled with `app: httpbin`:

```yaml
selector:
  matchLabels:
    app: httpbin
```

Apply the policy:

```bash
kubectl apply -f auth_policy.yaml
```

---

## Step 7: Deploy Hello World Application

Deploy the Hello World app:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/helloworld/helloworld.yaml
```

Test access:

```bash
kubectl exec -ti test -n test -- curl http://helloworld.default.svc.cluster.local:5000/hello
# Expected: Hello version: v1, instance: helloworld-v1-<pod>
```

---

## Summary

* **PeerAuthentication** enforces mTLS across the mesh
* **AuthorizationPolicy** allows granular control of requests by method, path, and namespace
* **DENY rules** take precedence over ALLOW rules
* Selectors enable policy targeting for specific workloads

---

This README ensures a **step-by-step guide** for anyone wanting to replicate the Istio Authorization lab.

---
