
# Istio Ingress Gateway Configuration Lab

This lab guides you through configuring **Ingress Gateways** within the **Istio Service Mesh** and exposing the Bookinfo sample application externally.

---

## 🧩 Prerequisites

* Kubernetes cluster
* Istio installed
* Automatic sidecar injection enabled in the `default` namespace

Verify injection:

```bash
kubectl get ns --show-labels
```

Expected output:

```
default           Active   ...   istio-injection=enabled
istio-system      Active   ...
...
```

---

# 1. Deploy the Bookinfo Application

Apply the Bookinfo sample manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/bookinfo/platform/kube/bookinfo.yaml
```

Verify services:

```bash
kubectl get svc
```

Verify pods and sidecar injection:

```bash
kubectl get po
```

Expected: **2/2 containers running** per pod (app + Envoy).

---

# 2. Create the Virtual Service (Initial Version)

Apply the initial Virtual Service:

```bash
kubectl apply -f virtualService.yaml
```

---

# 3. Identify Ingress Gateway Labels

We will need the label that identifies the ingress gateway for Gateway configuration.

```bash
kubectl describe pod -n istio-system istio-ingress | grep Labels -A 20
```

Key label:

```
istio=ingressgateway
```

---

# 4. Create the Gateway Resource

Create a Gateway in the `istio-system` namespace to expose HTTP traffic on **port 80** for host:

```
book.info.com
```

Apply your Gateway:

```bash
kubectl apply -f gateway.yaml
```

Verify:

```bash
kubectl get gateways -A
```

---

# 5. Verify Ingress Gateway Service (NodePort)

Retrieve the NodePort info:

```bash
kubectl get svc -n istio-system
```

Example output:

```
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
istio-ingressgateway   NodePort    10.98.54.69     <none>        80:32177/TCP ...
```

NodePort **IP** = cluster node IP or gateway service IP
NodePort **Port** = 32177 (from example)

---

# 6. Test External Access (Fails Initially)

```bash
curl --head --header "Host: book.info.com" http://10.98.54.69
```

Output:

```
HTTP/1.1 404 Not Found
```

### Why does this fail?

Because:

1. **Virtual Service only referenced `productpage` host**, not the external host.
2. It **did not specify `gateways:`**, so it only applies internally.

---

# 7. Fix Virtual Service (Add External Host + Gateway)

Update the Virtual Service to include:

```yaml
hosts:
  - book.info.com
gateways:
  - istio-gateway
```

Reapply:

```bash
kubectl apply -f virtualService.yaml
```

---

# 8. Test External Access Again (Success)

```bash
curl --head --header "Host: book.info.com" http://10.98.54.69
```

Expected:

```
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
server: istio-envoy
```

Congratulations!
Your Bookinfo application is now accessible via the **Istio Ingress Gateway**.

---

# ✅ Summary

You have successfully:

* Deployed Bookinfo with sidecar injection
* Created a Virtual Service
* Identified ingress gateway labels
* Created an Istio Gateway for external traffic
* Connected the Virtual Service to the Gateway
* Validated external access

