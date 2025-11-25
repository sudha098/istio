
# Istio Ambient – Traffic Management Lab

This lab walks you through:

* Observing why a 95/5 L7 traffic split **fails** in Ambient mode without a Waypoint.
* Enabling a **Waypoint proxy** to support proper Layer 7 traffic splitting.
* Migrating from `VirtualService`/`DestinationRule` to **Gateway API (HTTPRoute)**.
* Performing L7 traffic manipulation on `httpbin` (delay, abort) later in the lab.

---

## 🔧 Prerequisites

Two YAML files should exist in the current directory:

* `helloworld.yaml`
* `httpbin.yaml`

---

# 1. Deploy the HelloWorld App

```bash
kubectl apply -f helloworld.yaml -n hello
```

Verify resources:

```bash
kubectl get pods -n hello
kubectl get svc -n hello
```

Expected:

```
helloworld-v1-... Running
helloworld-v2-... Running

svc/helloworld 5000/TCP
```

---

# 2. Create a DestinationRule for v1/v2

```bash
kubectl apply -f hello-dr.yaml
kubectl get destinationrules -n hello
```

You should see:

```
NAME             HOST
hello-world-dr   helloworld
```

---

# 3. Create a VirtualService for a 95/5 Split

```bash
kubectl apply -f hellovs.yaml
kubectl get virtualservice -n hello
```

---

# 4. Test Traffic Distribution

Run multiple requests from the `curl` test pod:

```bash
for i in {1..10}; do \
  kubectl exec curl -n test -- curl -s helloworld.hello.svc.cluster.local:5000/hello; \
done
```

**Observed behavior:**
Traffic is roughly **50/50**, *not 95/5*.

Example output:

```
Hello version: v2 ...
Hello version: v1 ...
Hello version: v1 ...
Hello version: v2 ...
...
```

## ❗ Why does the 95/5 split fail?

Check namespace labels:

```bash
kubectl get ns --show-labels
```

Observation:
The **hello** namespace has **no Ambient or Waypoint labels**, meaning:

* Ambient mode: **off**
* Waypoint proxy: **not enabled**

➡️ **Without a Waypoint, Ambient mode cannot perform L7 traffic management.**
DestinationRules + VirtualServices have no L7 enforcement → only L4 load balancing → 50/50 round-robin.

---

# 5. Enable Ambient Mode + Waypoint for the Hello Namespace

```bash
kubectl label ns hello istio.io/dataplane-mode=ambient \
  istio.io/use-waypoint=waypoint --overwrite
```

Deploy the waypoint:

```bash
istioctl waypoint apply -n hello
kubectl get deploy waypoint -n hello
```

---

# 6. Remove Old Istio Config (VS/DR)

```bash
kubectl delete virtualservice hello-world-vs -n hello --ignore-not-found
kubectl delete destinationrule hello-world-dr -n hello --ignore-not-found
```

You will now transition to **Gateway API**.

---

# 7. Create an HTTPRoute for 95/5 Traffic Split

Verify CRD exists:

```bash
kubectl get crd httproutes.gateway.networking.k8s.io -o name
```

Apply the HTTPRoute:

```yaml
# hello-httproute-split-traffic.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-http-split-traffic
  namespace: hello
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: helloworld
    port: 5000
  rules:
  - backendRefs:
    - name: helloworld-v1
      port: 5000
      weight: 95
    - name: helloworld-v2
      port: 5000
      weight: 5
```

Apply:

```bash
kubectl apply -f hello-httproute-split-traffic.yaml
kubectl get httproutes -n hello
```

---

# ✔️ Summary

| Feature                                           | Without Waypoint | With Waypoint         |
| ------------------------------------------------- | ---------------- | --------------------- |
| L4 routing                                        | ✅ Works          | ✅ Works               |
| L7 routing (weighting, rewrites, fault injection) | ❌ Not supported  | ✅ Supported           |
| VirtualService / DR                               | Ignored          | Deprecated in Ambient |
| Gateway API HTTPRoute                             | Partially        | Fully supported       |

You have now correctly:

* Enabled Ambient + Waypoint
* Switched from outdated VS/DR to **Gateway API**
* Achieved proper **95/5 L7 traffic splitting**


