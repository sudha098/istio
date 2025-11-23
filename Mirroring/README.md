
# 🚀 Istio Traffic Mirroring Lab

Configure Traffic Mirroring in Istio Service Mesh

## 📌 Overview

This lab demonstrates how to configure **Traffic Mirroring** in the **Istio service mesh**.
You will:

* Deploy two versions of an Echo application
* Expose them through a Kubernetes Service
* Use a test client to generate traffic
* Configure **DestinationRule** and **VirtualService**
* Mirror 100% of production traffic (v1 → v2)
* Observe mirrored traffic in logs

---

## 🛠 Prerequisites

* A Kubernetes cluster with **Istio sidecar injection enabled** in the `default` namespace
* `kubectl` configured and authenticated
* Istio installed (minimal profile is sufficient)

Check sidecar injection:

```bash
kubectl get namespace default --show-labels
```

Expected:

```
istio-injection=enabled
```

---

## 📦 Step 1: Verify Echo Deployments

Two versions of the Echo server are already deployed:

```bash
kubectl get pods
```

Example output:

```
echo-server-v1-xxxx   2/2   Running
echo-server-v2-yyyy   2/2   Running
```

View pod labels:

```bash
kubectl get pods --show-labels
```

Both pods share:

```
app=echo-server
```

But have distinct versions:

```
version=v1
version=v2
```

---

## 🔧 Step 2: Create the Kubernetes Service

Create `echo_svc.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-server
  labels:
    app: echo-server
spec:
  selector:
    app: echo-server
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f echo_svc.yaml
```

Verify:

```bash
kubectl get svc --show-labels
```

---

## 🧪 Step 3: Test the Echo App

Create a test pod:

```bash
kubectl run test --image=nginx
```

Execute into the container:

```bash
kubectl exec -ti test -- /bin/bash
```

Send traffic:

```bash
curl -s http://echo-server \
 | grep -o '"HOSTNAME":"[^"]*"' \
 | sed 's/"HOSTNAME":"\(.*\)"/HOSTNAME: \1/'
```

You should observe **load balancing** between v1 and v2 (round-robin).

---

## 📡 Step 4: Enable Traffic Mirroring

### 1. Create DestinationRule

**destinationRules.yaml**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: echo-server
spec:
  host: echo-server
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

Apply:

```bash
kubectl apply -f destinationRules.yaml
```

---

### 2. Create VirtualService for Routing + Mirroring

**virtualService.yaml**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: echo-server
spec:
  hosts:
    - echo-server
  http:
    - route:
        - destination:
            host: echo-server
            subset: v1
          weight: 100
      mirror:
        host: echo-server
        subset: v2
      mirrorPercentage:
        value: 100.0
```

Apply:

```bash
kubectl apply -f virtualService.yaml
```

---

## 📊 Step 5: Observe Traffic Mirroring

Open **two terminals**:

Terminal 1:

```bash
kubectl logs -f echo-server-v1-xxxxx
```

Terminal 2:

```bash
kubectl logs -f echo-server-v2-yyyyy
```

Generate traffic:

```bash
kubectl exec -ti test -- /bin/bash
curl http://echo-server
```

### Expected Results

* **Responses always come from v1** (100% primary routing)
* **v2 logs receive mirrored requests** (shadow traffic)

Example:

**v1 log:**

```
[GET] - http://echo-server/
```

**v2 mirrored log:**

```
[GET] - http://echo-server-shadow/
```

---

## ✅ Result

Traffic mirroring is working:

* ✔ Production version (**v1**) receives 100% of actual traffic
* ✔ New version (**v2**) receives mirrored traffic for testing
* ✔ No impact on real users
* ✔ Safe testing of new features

---

## 📁 Repository Structure Example

```
.
├── echo_svc.yaml
├── destinationRules.yaml
├── virtualService.yaml
└── README.md
```

---

## 📚 Additional Notes

Traffic Mirroring is ideal for:

* Testing new releases
* Comparing request behavior
* Ensuring compatibility
* Performance validation

It allows teams to validate production traffic patterns **without impacting users**.

