Sure! Here's a polished GitHub README based on the lab steps you provided:

---

# Istio Authentication Lab

This lab demonstrates how to configure **authentication** in an Istio service mesh using **PeerAuthentication policies**. By the end of this lab, you will understand how to enforce **mTLS**, use **namespace-level overrides**, and apply **workload-specific policies**.

---

## Prerequisites

* Kubernetes cluster with Istio installed
* `kubectl` configured to access the cluster
* Istio sidecar injection enabled on the `default` namespace

Verify Istio injection:

```bash
kubectl get ns --show-labels
```

You should see:

```
NAME              STATUS   AGE    LABELS
default           Active   24m    istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   107s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   24m    kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   24m    kubernetes.io/metadata.name=kube-public
kube-system       Active   24m    kubernetes.io/metadata.name=kube-system
```

---

## Step 1: Deploy Hello World Application

Deploy the Hello World sample:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/helloworld/helloworld.yaml
```

Check the pods:

```bash
kubectl get po
```

Expected output:

```
NAME                             READY   STATUS    RESTARTS   AGE
helloworld-v1-...                2/2     Running   0          13s
helloworld-v2-...                2/2     Running   0          13s
```

---

## Step 2: Create Test Namespace and Pod

Create a `test` namespace and deploy an `nginx` pod:

```bash
kubectl create ns test
kubectl run test --image=nginx -n test
kubectl get po -n test
```

---

## Step 3: Test Connectivity to Hello World

Check that the Hello World service is accessible:

```bash
kubectl exec -ti -n test test -- curl helloworld.default.svc.cluster.local:5000/hello
```

Expected output:

```
Hello version: v2, instance: helloworld-v2-...
```

---

## Step 4: Enforce Global mTLS

Create a global **PeerAuthentication** policy to enforce **STRICT mTLS**:

```bash
kubectl apply -f peer_auth_global.yaml
```

Verify connectivity from `test` pod fails:

```bash
kubectl exec -ti -n test test -- curl --head http://helloworld.default.svc:5000/hello
```

Expected output:

```
curl: (56) Recv failure: Connection reset by peer
```

This happens because the `test` namespace does not have Istio injection enabled, so traffic is plaintext.

---

## Step 5: Enable Istio Injection on Test Namespace

```bash
kubectl label ns test istio-injection=enabled
kubectl delete po test -n test
kubectl run test --image=nginx -n test
```

Test connectivity again:

```bash
kubectl exec -ti -n test test -- curl --head http://helloworld.default.svc:5000/hello
```

Expected output:

```
HTTP/1.1 200 OK
server: envoy
...
```

---

## Step 6: Namespace-level Override

Remove Istio injection from `test` namespace:

```bash
kubectl label ns test istio-injection-
kubectl exec -ti -n test test -- curl --head http://helloworld.default.svc:5000/hello
```

Expected output:

```
curl: (56) Recv failure: Connection reset by peer
```

Create a **namespace-level PERMISSIVE policy** in `default`:

```bash
kubectl apply -f peer_auth_default.yaml
kubectl exec -ti -n test test -- curl --head http://helloworld.default.svc:5000/hello
```

Expected output:

```
HTTP/1.1 200 OK
server: istio-envoy
...
```

---

## Step 7: Workload-specific Policy

Restrict PERMISSIVE mode to only the Hello World workload by adding a selector to the PeerAuthentication policy:

```bash
kubectl apply -f peer_auth_default.yaml
```

---

## Step 8: Deploy BookInfo App

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/bookinfo/platform/kube/bookinfo.yaml
```

Test connectivity from `test` pod:

* **BookInfo app** (should fail):

```bash
kubectl exec -ti -n test test -- curl --head http://productpage.default.svc:9080/productpage
```

* **Hello World app** (should succeed):

```bash
kubectl exec -ti -n test test -- curl --head http://helloworld.default.svc:5000/hello
```

---

## ✅ Summary

* Deployed Hello World and BookInfo applications in Istio service mesh
* Enforced **global STRICT mTLS**
* Demonstrated **namespace-level override** using PERMISSIVE mode
* Applied **workload-specific PeerAuthentication** policies

This lab shows how Istio can provide **fine-grained authentication control** over services in a Kubernetes cluster.

---
