
# 🌐 Istio on GKE – Demo Guide

This guide demonstrates **Istio traffic management, Virtual Services, Destination Rules, and Sidecars** deployed on **Google Kubernetes Engine (GKE)**.

---

## ⚙️ Prerequisites

* GKE cluster with Istio installed
* kubectl configured for your cluster
* Istio CLI (`istioctl`) installed

---

## 🚀 Deployment Steps

### 1️⃣ Verify Istio-Enabled Namespaces

```bash
kubectl get ns --show-labels
```

Expected output:

```
default      Active   istio-injection=enabled
istio-system Active
```

---

### 2️⃣ Deploy Sample Applications

#### HTTPBin

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml
kubectl get pods
```

#### HelloWorld

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/helloworld/helloworld.yaml
kubectl get pods
```

---

### 3️⃣ Create Test Namespace & Pods

```bash
kubectl create ns test
kubectl run test --image=nginx -n test
kubectl run nginx --image=nginx -n test
```

Enable Istio injection for the test namespace:

```bash
kubectl label namespace test istio-injection=enabled
kubectl delete pod -n test test
kubectl delete pod -n test nginx
kubectl run test --image=nginx -n test
kubectl run nginx --image=nginx -n test
```

---

### 4️⃣ Apply Istio Resources

Apply Virtual Services, Destination Rules, Sidecars, and PeerAuthentication as required.

---

### 5️⃣ Test Connectivity

```bash
kubectl exec -ti -n test test -- curl --head httpbin.default.svc.cluster.local:8000
kubectl exec -ti -n test test -- curl helloworld.default.svc.cluster.local:5000/hello
```

---

## 📊 Architecture Diagrams (GKE + Istio)

### 1️⃣ Istio High-Level Flow (Pods with & without Sidecar)

```mermaid
flowchart TD
    A[Test Pod no sidecar] -->|Bypasses Mesh| B[HTTP Service]
    B --> C[Direct ClusterIP Access]

    D[Test Pod with sidecar] -->|Intercepted by Envoy| E[VirtualService Rules]
    E -->|Route| B
```

---

### 2️⃣ Virtual Service Routing Logic

```mermaid
flowchart TD
    A[Test Pod - Sidecar Enabled] --> B[Envoy Proxy]
    B --> C[VirtualService: httpbin]
    C --> D[Destination: httpbin.default.svc:8000]
    D --> E[httpbin Deployment]
```

✅ This **renders correctly on GitHub**.

If you want, I can **rewrite all your Istio Virtual Service README diagrams** in this fully GitHub-compatible format so every diagram works properly.

Do you want me to do that?

```

---

### 3️⃣ Sidecar Traffic Control

```mermaid
flowchart TD
    A[Pod with Sidecar] --> B[Envoy]
    B --> C[Sidecar Config Limits Outgoing Traffic]
    C --> D[Allowed Services]
```

---

### 4️⃣ GKE Cluster + Istio Overview

```mermaid
flowchart TD
    subgraph GKE Cluster
        A[Kubernetes Node 1]
        B[Kubernetes Node 2]
        C[Kubernetes Node 3]
    end

    A -->|Istio Sidecar| D[Pod A]
    B -->|Istio Sidecar| E[Pod B]
    C -->|Istio Sidecar| F[Pod C]

    D -->|Traffic routed via| G[Envoy]
    E -->|Traffic routed via| G
    F -->|Traffic routed via| G
```

