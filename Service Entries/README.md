
# Istio Service Mesh Lab: ServiceEntry & Egress Gateway Configuration

This lab walks through configuring **Istio outbound traffic control**, using **ServiceEntry** and **Egress Gateway** resources to manage external access (e.g., wikipedia.org) from workloads inside a Kubernetes cluster.

You will:

* Install Istio 1.26.0
* Configure the mesh to use **REGISTRY_ONLY** outbound traffic policy
* Validate traffic behavior before and after enabling Istio sidecar injection
* Create **ServiceEntry** to allow controlled access to Wikipedia
* Configure an **Egress Gateway**
* Route Wikipedia traffic through the Egress Gateway and verify logs

---

## 1. Verify Kubernetes Is Operational

```bash
kubectl get pods -A
```

Ensure that all core Kubernetes components (coredns, kube-proxy, etc.) are running.

---

## 2. Download & Install Istio 1.26.0

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.26.0 sh -
cp /root/istio-1.26.0/bin/istioctl /usr/local/bin/
```

Verify Istio client:

```bash
istioctl version
```

---

## 3. Install Istio Using Demo Profile With OutboundTrafficPolicy=REGISTRY_ONLY

```bash
istioctl install --set profile=demo \
  --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
```

Confirm mesh configuration:

```bash
kubectl get configmap istio -n istio-system -o yaml | grep outboundTrafficPolicy -A 2
```

Expected:

```
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
```

---

## 4. Validate Istio Installation

```bash
kubectl get pods -n istio-system
```

Expected:

```
istio-egressgateway (...)
istio-ingressgateway (...)
istiod (...)
```

---

## 5. Deploy Test Pod (Without Sidecar Injection)

```bash
kubectl run test --image=nginx
kubectl exec -ti test -- curl --head -L http://www.wikipedia.org
```

Result: **200 OK**
Reason: The default namespace has no sidecar → traffic bypasses Istio.

---

## 6. Enable Istio Sidecar Injection in `default` Namespace

```bash
kubectl label ns default istio-injection=enabled
kubectl delete pod test
kubectl run test --image=nginx
```

Test again:

```bash
kubectl exec -ti test -- curl --head -L http://www.wikipedia.org
```

Expected: **502 Bad Gateway**

### ❗ Why 502?

Because **REGISTRY_ONLY** mode blocks all outbound traffic unless explicitly defined by a ServiceEntry.

---

## 7. Create ServiceEntry for Wikipedia

`service_entry.yaml` example:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: wikipedia-egress
spec:
  hosts:
  - www.wikipedia.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
```

Apply:

```bash
kubectl apply -f service_entry.yaml
```

Test:

```bash
kubectl exec -ti test -- curl --head -L http://www.wikipedia.org
```

Result: **Works**
Other domains (e.g., google.com) still blocked because they are not defined.

---

## 8. Why Google Still Returns 502?

A **502 Bad Gateway** means:

> Istio Envoy attempted to route traffic but found **no valid cluster**, because outbound traffic is restricted to REGISTRY_ONLY and google.com has **no ServiceEntry**.

---

## 9. Inspect Egress Gateway Logs

Find the pod:

```bash
kubectl get pod -n istio-system
```

Tail logs:

```bash
kubectl logs -f -n istio-system <egressgateway-pod>
```

Make requests:

```bash
kubectl exec -ti test -- curl --head -L http://www.google.com   # 502, no logs
kubectl exec -ti test -- curl --head -L http://www.wikipedia.org # works, no logs yet
```

No logs appear because **traffic is not routed through the Egress Gateway yet** — the ServiceEntry only enables **direct** egress.

---

## 10. Route Wikipedia Traffic Through the Egress Gateway

You must create:

### 10.1 Gateway (`istio-egressgateway`)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - www.wikipedia.org
```

### 10.2 DestinationRule (`egressgateway-for-wikipedia`)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: egressgateway-for-wikipedia
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: wikipedia
```

### 10.3 VirtualService (`wikipedia-egress-gateway`)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: wikipedia-egress-gateway
spec:
  hosts:
  - www.wikipedia.org
  gateways:
  - mesh
  - istio-egressgateway
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: wikipedia
        port:
          number: 80
  - match:
    - gateways:
      - istio-egressgateway
      port: 80
    route:
    - destination:
        host: www.wikipedia.org
        port:
          number: 80
```

Apply all resources:

```bash
kubectl apply -f egressgateway.yaml
kubectl apply -f virtualService.yaml
kubectl apply -f destinationRules.yaml
```

---

## 11. Verify Routing Through the Egress Gateway

Tail logs:

```bash
kubectl logs -f -n istio-system <egressgateway-pod>
```

Send request:

```bash
kubectl exec -ti test -- curl --head -L http://www.wikipedia.org
```

Now you **should see logs** in the Egress Gateway — confirming that traffic flows through it.

---

## Summary of Expected Behavior

| Scenario                  | Wikipedia            | Google    | Reason                             |
| ------------------------- | -------------------- | --------- | ---------------------------------- |
| No sidecar                | ✔ Allowed            | ✔ Allowed | No Istio enforcement               |
| Sidecar + REGISTRY_ONLY   | ❌ 502                | ❌ 502     | Blocked unless ServiceEntry exists |
| ServiceEntry only         | ✔ Allowed            | ❌ 502     | Wikipedia registered, Google not   |
| Egress Gateway configured | ✔ Routed via Gateway | ❌ 502     | Only Wikipedia allowed & routed    |

---

## Final Notes

This lab demonstrates:

* How **REGISTRY_ONLY** mode strictly restricts traffic
* How **ServiceEntry** restores explicit external connectivity
* How to route traffic through **Egress Gateways** for auditing, policy enforcement, or firewall integration
* How to verify traffic paths using **proxy logging**



Just tell me!
