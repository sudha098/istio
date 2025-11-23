
# Bringing an External Host into the Istio Mesh

### Using a ServiceEntry + VirtualService + Sidecar Injection

This lab demonstrates how to bring an external service (`myapp.com`) into the Istio service mesh using **ServiceEntry** and **VirtualService**, and how to validate traffic flow from a test workload.

---

## ✅ 1. Verify Istio Installation

List Istio control-plane pods:

```bash
kubectl get pods -n istio-system
```

Expected output:

```
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-675cdb9f4b-qngtv    1/1     Running   0          2m6s
istio-ingressgateway-6cd9bc7f5b-w2z79   1/1     Running   0          2m6s
istiod-7f898458c5-9dtdn                 1/1     Running   0          2m14s
```

---

## ✅ 2. Validate Local Web Server Accessibility

The host has an Nginx server accessible via DNS `myapp.com`.

### Test connectivity from the host:

```bash
curl myapp.com
```

Expected Nginx welcome page output.

### Validate DNS resolution:

```bash
ping myapp.com
```

Example:

```
PING myapp.com (192.168.121.254)
```

Take note of this **IP address**, you will use it in the `ServiceEntry`.

---

## ✅ 3. Create the ServiceEntry

Create a file `service_entry.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: myapp-service-entry
spec:
  hosts:
  - myapp.com
  location: MESH_EXTERNAL
  resolution: STATIC
  endpoints:
  - address: 192.168.121.254   # ← replace with your observed IP
    ports:
      http: 80
  ports:
  - number: 80
    name: http
    protocol: HTTP
```

Apply it:

```bash
kubectl apply -f service_entry.yaml
```

Confirm:

```bash
kubectl get serviceentries
```

---

## ✅ 4. Create a Test Pod

```bash
kubectl run test --image=nginx
```

---

## ❗ 5. Attempt Curl From Test Pod (Fails)

```bash
kubectl exec -ti test -- curl myapp.com
```

Result:

```
302 Found
```

This happens because:

### **Why 302 Happens**

* The **test pod did NOT have a sidecar injected**.
* Traffic **did not go through Envoy**, so the VirtualService routing was *not applied*.
* With only the ServiceEntry active, outbound traffic bypasses Istio and reaches a network gateway, resulting in a **302 redirect** (from your local network/gateway `stgw`).

---

## ✅ 6. Enable Sidecar Injection in Default Namespace

```bash
kubectl label ns default istio-injection=enabled
```

Recreate the test pod:

```bash
kubectl delete pod test
kubectl run test --image=nginx
```

Verify the pod now contains **2/2 containers**:

```bash
kubectl get pods
```

---

## ✅ 7. Create the Virtual Service

Create `virtualService.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nginx-local-vs
spec:
  hosts:
  - myapp.com
  http:
  - route:
    - destination:
        host: myapp.com
        port:
          number: 80
```

Apply it:

```bash
kubectl apply -f virtualService.yaml
```

Check:

```bash
kubectl get vs
```

---

## ✅ 8. Test Again (Now Works)

```bash
kubectl exec test -- curl myapp.com
```

Expected output: full Nginx welcome page.

---

# 🎉 Result

You have successfully:

* Added an external service into the Istio mesh using **ServiceEntry**
* Routed traffic through the mesh using **VirtualService**
* Ensured correct routing by enabling **sidecar injection**
* Validated working traffic flow from a workload running inside the cluster


