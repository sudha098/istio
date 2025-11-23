# Istio Lab: Timeouts & Retry Policies

This lab demonstrates how to configure **request timeouts** and **retry behavior** in the **Istio Service Mesh** using the `httpbin` workload.

---

## 1. Verify Istio Sidecar Injection

Ensure that the **default** namespace has automatic Istio sidecar injection enabled.

```bash
kubectl get ns --show-labels
```

Expected output:

```
NAME              STATUS   AGE   LABELS
default           Active   20m   istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   79s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   20m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   20m   kubernetes.io/metadata.name=kube-public
kube-system       Active   20m   kubernetes.io/metadata.name=kube-system
```

---

## 2. Deploy the HTTPBin Application

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml
```

Verify:

```bash
kubectl get pods
```

Example output:

```
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-686d6fc899-b9dht   2/2     Running   0          56s
```

---

## 3. Deploy a Test Client Pod

```bash
kubectl run test --image=nginx
kubectl get po
```

Example:

```
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-686d6fc899-8hx55   2/2     Running   0          26s
test                       2/2     Running   0          17s
```

---

## 4. Validate the httpbin Service

```bash
kubectl get svc
```

Expected:

```
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
httpbin    ClusterIP   10.xxx.xxx.xxx   <none>        8000/TCP   3m
```

---

## 5. Test the HTTPBin `/delay` Endpoint

The httpbin application provides:

```
/delay/<seconds>
```

Example test:

```bash
kubectl exec -ti test -- curl --head http://httpbin.default.svc:8000/delay/5
```

Expected successful response:

```
HTTP/1.1 200 OK
server-timing: initial_delay;dur=5000.00
x-envoy-upstream-service-time: 5006
```

At this point the application delays but still responds normally.

---

# Implementing Istio Timeouts

---

## 6. Create a VirtualService with a 2-Second Timeout

Create **virtualservice.yaml**:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-vs
spec:
  hosts:
    - httpbin
  http:
    - timeout: 2s
      route:
        - destination:
            host: httpbin
            port:
              number: 8000
```

Apply:

```bash
kubectl apply -f virtualservice.yaml
kubectl get vs
```

Expected:

```
NAME         GATEWAYS   HOSTS         AGE
httpbin-vs              ["httpbin"]   0s
```

This configuration enforces that **any request taking longer than 2 seconds will fail with a timeout**.

---

## 7. Test Timeout Behavior

### Request that completes within 2 seconds

```bash
kubectl exec -ti test -- curl --head http://httpbin.default.svc:8000/delay/1
```

Expected **200 OK**:

```
HTTP/1.1 200 OK
server-timing: initial_delay;dur=1000.00
x-envoy-upstream-service-time: 1002
```

### Request that exceeds timeout limit

```bash
kubectl exec -ti test -- curl --head http://httpbin.default.svc:8000/delay/5
```

Expected **504 Gateway Timeout**:

```
HTTP/1.1 504 Gateway Timeout
content-length: 24
content-type: text/plain
server: envoy
```

This confirms the timeout rule is being enforced by Istio.

---

# (Optional) Retry Policies

If you'd like, I can add a complete section demonstrating:
✔️ exponential retry delays
✔️ per-try timeouts
✔️ retry budgets
✔️ visual verification using Envoy access logs

---

## 🎉 Lab Complete

You successfully configured:

✔️ Service deployment
✔️ Routing through Istio sidecars
✔️ HTTP timeout policies
✔️ Validation of timeout behavior



