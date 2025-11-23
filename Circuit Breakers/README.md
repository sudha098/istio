# 🚦 Istio Circuit Breaking Lab

Configure and Validate Circuit Breakers in the Istio Service Mesh

## 📌 Overview

This lab demonstrates how to configure **Circuit Breaking** in Istio using:

* `VirtualService`
* `DestinationRule`
* `Fortio` for load testing
* Istio sidecar (Envoy) metrics for validation

You will simulate traffic overload and observe how Istio protects your services by rejecting excess requests and ejecting unhealthy hosts.

---

# 1️⃣ Verify Istio Sidecar Injection

Ensure the **default namespace** has automatic sidecar injection enabled:

```bash
kubectl get ns --show-labels
```

Expected:

```
default  ...  istio-injection=enabled
```

If not enabled:

```bash
kubectl label namespace default istio-injection=enabled
```

---

# 2️⃣ Deploy the Echo Server

Create `echo_deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
        - name: echo-server
          image: ealen/echo-server
          ports:
            - containerPort: 80
```

Create `echo_svc.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-server
spec:
  selector:
    app: echo-server
  ports:
    - port: 80
      targetPort: 80
```

Apply both:

```bash
kubectl apply -f echo_deployment.yaml -f echo_svc.yaml
```

Verify:

```bash
kubectl get po
```

---

# 3️⃣ Deploy Fortio Load Generator

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.25/samples/httpbin/sample-client/fortio-deploy.yaml
```

Check pods:

```bash
kubectl get po
```

Expected:

```
echo-server-xxxx      2/2 Running
fortio-deploy-xxxx    2/2 Running
```

---

# 4️⃣ Test Basic Connectivity

```bash
kubectl exec fortio-deploy-xxxx -c fortio -- \
  /usr/bin/fortio curl -quiet http://echo-server | grep -o '"HOSTNAME":"[^"]*"'
```

Expected output: returns pod hostname (200 OK).

---

# 5️⃣ Create a VirtualService

Create `echo_vs.yaml`:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: echo-vs
spec:
  hosts:
    - echo-server
  http:
    - route:
        - destination:
            host: echo-server
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f echo_vs.yaml
```

---

# 6️⃣ Create a DestinationRule with Circuit Breaking

Create `echo-dr.yaml`:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: echo-dr
spec:
  host: echo-server
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 5s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
```

Apply:

```bash
kubectl apply -f echo-dr.yaml
```

---

# 7️⃣ Generate Traffic (Light Load)

Run 2 concurrent threads, 20 total requests:

```bash
kubectl exec fortio-deploy-xxxx -c fortio -- \
  /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning http://echo-server
```

You should see **a few 503 errors** (circuit breaking).

---

# 8️⃣ Increase Load (10 Threads)

```bash
kubectl exec fortio-deploy-xxxx -c fortio -- \
  /usr/bin/fortio load -c 10 -qps 0 -n 40 -loglevel Warning http://echo-server
```

Expected: **~85% 503 errors** due to:

* maxConnections = 1
* max pending requests = 1

The circuit breaker rejects excess traffic as designed.

---

# 9️⃣ Inspect Envoy Proxy Stats

```bash
kubectl exec fortio-deploy-xxxx -c istio-proxy -- \
  pilot-agent request GET stats | grep echo-server | grep pending
```

Example fields:

* `.upstream_rq_pending_overflow` → **requests rejected due to circuit break**
* `.circuit_breakers.default.remaining_pending` → pending request limit
* `.upstream_rq_pending_total` → total pending requests

---

# 🔟 Adjust the Circuit Breaker (Optional)

Increase allowed connections to reduce failures.

Modify in `echo-dr.yaml`:

```yaml
connectionPool:
  tcp:
    maxConnections: 10
```

Apply:

```bash
kubectl apply -f echo-dr.yaml
```

Test again:

```bash
kubectl exec fortio-deploy-xxxx -c fortio -- \
  /usr/bin/fortio load -c 2 -qps 0 -n 40 -loglevel Warning http://echo-server
```

Expected: **significant drop in 503 errors**.

---

# ✔️ Summary

In this lab, you learned how to:

### ✅ Deploy a service behind Istio sidecars

### ✅ Apply VirtualServices and DestinationRules

### ✅ Configure advanced circuit breaking rules (TCP + HTTP)

### ✅ Use Fortio to simulate realistic load

### ✅ Inspect Envoy proxy metrics

### ✅ Observe overload protection in action

Circuit breakers help ensure application resilience by:

* Preventing cascading failures
* Rejecting excess load
* Ejecting unhealthy hosts
* Stabilizing overloaded systems

---

Great — here is a clean **Istio Circuit Breaker Architecture Diagram (text-based)** *plus* a set of **realistic Chaos Engineering scenarios** you can run to validate the circuit-breaker behavior.

---

# ✅ **1. Architecture Diagram for Istio Circuit Breaking (ASCII Diagram)**

```
                               ┌────────────────────────────┐
                               │        Fortio Client       │
                               │  (Load Generator Pod)      │
                               └───────────┬────────────────┘
                                           │
                                           ▼
                               ┌────────────────────────────┐
                               │ fortio Sidecar Proxy       │
                               │ (Envoy)                    │
                               └───────┬────────────────────┘
                                       │  outbound traffic
                                       ▼
                       ┌────────────────────────────────────────┐
                       │ Istio VirtualService: echo-vs          │
                       │ - Routes traffic to echo-server         │
                       └───────────────────┬────────────────────┘
                                           │
                                           ▼
                       ┌────────────────────────────────────────┐
                       │ DestinationRule: echo-dr               │
                       │ circuit_breakers:                      │
                       │   maxConnections: 1 (or 10 later)      │
                       │   http1MaxPendingRequests: 1           │
                       │   maxRequestsPerConnection: 1          │
                       │   outlierDetection:                    │
                       │     consecutive5xxErrors: 1            │
                       │     interval: 5s                       │
                       │     baseEjectionTime: 30s              │
                       └───────────────────┬────────────────────┘
                                           │
                                           ▼
                 ┌────────────────────────────────────────────────────┐
                 │ echo-server Service (ClusterIP)                   │
                 └───────────────────┬────────────────────────────────┘
                                     │
                                     ▼
                     ┌────────────────────────────────────┐
                     │ echo-server Pod                    │
                     │ ealen/echo-server image            │
                     │ Sidecar Proxy (Envoy)              │
                     └────────────────────────────────────┘

```

### Key Flow:

1. **Fortio** sends traffic → through **fortio’s Envoy proxy**
2. Envoy applies:

   * **VirtualService routing**
   * **DestinationRule circuit breakers**
3. Traffic reaches echo-server’s Envoy sidecar → echo-server pod.

---

# ✅ **2. Chaos Engineering Scenarios to Validate Circuit Breakers**

Below are high-value scenarios you can run to validate your mesh resilience.

---

# 🔥 **Chaos Scenario 1 — Sudden Traffic Surge (Load Spike)**

**Goal:** Confirm circuit breaker rejects extra traffic instead of overloading the service.

### How to run:

```
kubectl exec fortio-deploy-xxxx -c fortio -- \
  fortio load -c 50 -qps 0 -n 200 http://echo-server
```

### Expected Outcome:

* Many **503 UC (upstream overflow)** responses
* Envoy stats will show:

  * `upstream_rq_pending_overflow` > 0
* Echo-server pod stays **healthy** (no crashes)

---

# 🔥 **Chaos Scenario 2 — Kill the Echo Pod (Pod Crash Test)**

**Goal:** Verify outlier detection ejects the unhealthy host.

### Step 1: Delete the pod

```
kubectl delete pod -l app=echo-server
```

### Step 2: Immediately run:

```
fortio curl http://echo-server
```

### Expected:

* Some **503** while the pod restarts
* DR outlier detection sees 5xx → **ejects pod for 30 seconds**
* Fortio should show:

  * `upstream_rq_5xx`
  * `outlier_detection.ejections_total` increasing

---

# 🔥 **Chaos Scenario 3 — Introduce Latency (Network Delay)**

**Goal:** Validate that slow pods trigger ejection or degrade gracefully.

### Use `tc` to inject latency:

```
kubectl exec -it <echo-pod> -c echo-server -- \
  tc qdisc add dev eth0 root netem delay 500ms
```

Test:

```
fortio load -c 5 -n 50 http://echo-server
```

### Expected:

* Increased `upstream_rq_pending`
* Eventually → 5xx → ejection
* Requests routed away from this pod for 30s

---

# 🔥 **Chaos Scenario 4 — Block Echo-Service Port (Network Blackhole)**

**Goal:** See if Envoy detects connection failures and applies ejection.

### Inject network loss:

```
kubectl exec -it <echo-pod> -- \
  tc qdisc add dev eth0 root netem loss 100%
```

Run tests:

```
fortio load -c 5 -n 20 http://echo-server
```

### Expected Result:

* Many 503s
* Outlier ejection becomes active

---

# 🔥 **Chaos Scenario 5 — CPU Exhaustion on Echo Pod**

**Goal:** Verify circuit breaker activates when pod becomes slow or unresponsive.

### Apply stress:

```
kubectl exec -it <echo-pod> -c echo-server -- \
  stress --cpu 2 --timeout 60
```

Test:

```
fortio load -c 10 -n 100 http://echo-server
```

### Expected:

* Slow responses → 5xx → pod ejected
* Mesh avoids cascading failure

---

# 🔥 **Chaos Scenario 6 — Disable Sidecar in One Pod (Misconfiguration)**

**Goal:** Validate misconfigured pods are automatically bypassed.

### Patch deployment to remove the sidecar from one replica:

```
kubectl patch deploy echo-server \
  -p '{"spec": {"template": {"metadata": {"labels": {"istio-injection": "disabled"}}}}}'
```

Scale:

```
kubectl scale deploy echo-server --replicas=2
```

### Expected:

* Non-injected pod is considered **unhealthy/unreachable**
* Circuit breaker avoids routing to it

---

# 🔥 **Chaos Scenario 7 — DNS Failure for Echo-Service**

**Goal:** Validate Envoy handles DNS lookup issues gracefully.

Simulate:

```
kubectl exec -it <fortio-pod> -c istio-proxy -- \
  sed -i 's/kube-dns/kube-dns-fake/g' /etc/resolv.conf
```

### Expected:

* Requests fail—but circuit breaker metrics indicate failures gracefully
* No crashes, mesh remains up

---

# 📌 **Summary Table**

| Scenario          | Validates                        | Expected Behavior            |
| ----------------- | -------------------------------- | ---------------------------- |
| Load spike        | maxConnections, pending requests | 503 overflow, no pod crashes |
| Kill pod          | outlier detection                | Pod ejected for 30s          |
| Latency injection | slow host ejection               | fallback to healthy hosts    |
| Network blackhole | connection failure ejection      | avoid blackholed pod         |
| CPU stress        | timeout → 5xx → ejection         | graceful degradation         |
| Disable sidecar   | mesh consistency                 | misconfigured pod excluded   |
| DNS failure       | Envoy resiliency                 | controlled failure           |

---



