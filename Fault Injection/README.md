
# Istio Fault Injection Lab

This lab demonstrates how to configure **delay** and **abort** fault injection in the **Istio Service Mesh** using a simple **Hello World** application. You will deploy services, apply virtual services with fault rules, and verify their behavior.

---

## 1. Verify Istio sidecar injection

Ensure that the **default** namespace has automatic sidecar injection enabled:

```bash
kubectl get ns --show-labels
```

Expected output:

```
NAME              STATUS   AGE   LABELS
default           Active   23m   istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   87s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   23m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   23m   kubernetes.io/metadata.name=kube-public
kube-system       Active   23m   kubernetes.io/metadata.name=kube-system
```

---

## 2. Deploy the Hello World Application

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/helloworld/helloworld.yaml
```

Verify pods and service:

```bash
kubectl get po,svc
```

Example output:

```
NAME                                 READY   STATUS    RESTARTS   AGE
pod/helloworld-v1-5787f49bd8-vjssn   2/2     Running   0          81s
pod/helloworld-v2-6746879bdd-qqlcz   2/2     Running   0          81s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/helloworld   ClusterIP   10.104.75.149   <none>        5000/TCP   81s
```

---

## 3. Create a Test Pod

```bash
kubectl run test --image=nginx
kubectl get po
```

---

## 4. Verify Access to Hello World Service

```bash
kubectl exec -ti test -- curl http://helloworld.default.svc:5000/hello
```

Example output:

```
Hello version: v2, instance: helloworld-v2-6746879bdd-qqlcz
```

---

# Fault Injection

---

## 5. Create a VirtualService With 5-Second Delay

Create **virtual_service.yaml**:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: hello-world-vs
spec:
  hosts:
    - helloworld
  http:
    - fault:
        delay:
          fixedDelay: 5s
          percentage:
            value: 100.0
      route:
        - destination:
            host: helloworld
            port:
              number: 5000
```

Apply it:

```bash
kubectl apply -f virtual_service.yaml
kubectl get vs
```

Test the delay:

```bash
kubectl exec -ti test -- curl http://helloworld.default.svc:5000/hello
```

You should experience a **5-second delay** before each response.

---

## 6. Inject a 100% HTTP 500 Abort

Update the VirtualService:

```yaml
fault:
  abort:
    percentage:
      value: 100.0
    httpStatus: 500
```

Apply the update:

```bash
kubectl apply -f virtual_service.yaml
```

Test:

```bash
kubectl exec -ti test -- curl --head http://helloworld.default.svc:5000/hello
```

Expected:

```
HTTP/1.1 500 Internal Server Error
```

---

## 7. Inject a 50% HTTP 404 Failure

Update the VirtualService:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: hello-world-vs
spec:
  hosts:
    - helloworld
  http:
    - fault:
        abort:
          percentage:
            value: 50.0
          httpStatus: 404
      route:
        - destination:
            host: helloworld
            port:
              number: 5000
```

Apply:

```bash
kubectl apply -f virtual_service.yaml
```

Test multiple requests:

```bash
kubectl exec -ti test -- /bin/sh -c 'for i in $(seq 1 10); do curl --head http://helloworld.default.svc:5000/hello; echo "---"; done'
```

Example output:

```
HTTP/1.1 404 Not Found
---
HTTP/1.1 404 Not Found
---
HTTP/1.1 200 OK
---
HTTP/1.1 200 OK
---
HTTP/1.1 404 Not Found
---
...
```

You should see **approximately half 200 OK** and **half 404 Not Found**, depending on randomness.

---

## 🎉 Lab Completed

You have successfully implemented and validated:

✔️ Delay injection
✔️ Full abort injection
✔️ Partial abort injection
✔️ Testing with a client pod
✔️ Verifying Envoy-side behavior


