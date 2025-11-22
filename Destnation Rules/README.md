
# Destination Rules Lab

Welcome to the **Destination Rules Lab**. By the end of this lab, you will learn how to configure **Destination Rules** and **Virtual Services** in the Istio Service Mesh.

---

## 1. Verify Namespace Labels

```bash
kubectl get ns --show-labels
```

Example output:

```
NAME              STATUS   AGE     LABELS
default           Active   34m     istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   2m52s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   34m     kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   34m     kubernetes.io/metadata.name=kube-public
kube-system       Active   34m     kubernetes.io/metadata.name=kube-system
```

---

## 2. Deploy the Hello World Application

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/helloworld/helloworld.yaml
```

Check deployments:

```bash
kubectl get deployments.apps
```

```
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-v1   1/1     1            1           21s
helloworld-v2   1/1     1            1           21s
```

Check services:

```bash
kubectl get svc
```

```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.107.100.199   <none>        5000/TCP   34s
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    36m
```

---

## 3. Verify Sidecar Injection

Both pods must show `2/2` containers running:

```bash
kubectl get pods
```

```
NAME                             READY   STATUS    RESTARTS   AGE
helloworld-v1-5787f49bd8-bhwp7   2/2     Running   0          67s
helloworld-v2-6746879bdd-5nfpm   2/2     Running   0          67s
```

---

## 4. Create a Test Namespace and Pod

```bash
kubectl create ns test --dry-run=client -o yaml > test_ns.yaml
kubectl apply -f test_ns.yaml

kubectl run test --image=nginx -n test --dry-run=client -o yaml > test_pod.yaml
kubectl apply -f test_pod.yaml
```

Confirm injection:

```bash
kubectl get ns test --show-labels
kubectl get po -n test
```

```
NAME   READY   STATUS    RESTARTS   AGE
test   2/2     Running   0          11s
```

---

## 5. Verify Hello World Service

```bash
kubectl get svc
```

```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.107.100.199   <none>        5000/TCP   4m37s
```

---

## 6. Test Connectivity from the Test Pod

```bash
kubectl exec -ti -n test test -- curl helloworld.default.svc.cluster.local:5000/hello
```

Example responses:

```
Hello version: v2, instance: helloworld-v2-6746879bdd-5nfpm
Hello version: v1, instance: helloworld-v1-5787f49bd8-bhwp7
```

---

## 7. Review Pod Labels

```bash
kubectl get pods --show-labels
```

This confirms that labels such as `version: v1` and `version: v2` are applied and used by Istio.

---

## 8. Export the Hello World Service Definition

```bash
kubectl get svc helloworld -o yaml > helloWorldSvc.yaml
```

> **Note:**
> The service route is determined **only by the selector field**.
> Other labels (such as `service: helloworld`) do **not** affect routing.

---

## 9. Apply the Destination Rule

```bash
kubectl apply -f destinationRules.yaml
```

Verify:

```bash
kubectl get destinationrules
```

```
NAME             HOST         AGE
hello-world-ds   helloworld   59s
```

---

## 10. Apply the Virtual Service

```bash
kubectl apply -f virtualService.yaml
```

Check Virtual Service:

```bash
kubectl get vs
```

```
NAME             GATEWAYS   HOSTS            AGE
hello-world-vs              ["helloworld"]   5s
```

---

## 11. Test Traffic Splitting (Default)

Execute multiple curl commands:

```
curl helloworld.default.svc.cluster.local:5000/hello
```

You should see alternating responses from **v1** and **v2**.

Example:

```
Hello version: v1...
Hello version: v2...
Hello version: v1...
Hello version: v2...
```

---

## 12. Update Virtual Service for 90/10 Traffic Split

Modify the Virtual Service to send:

* **90% → v1**
* **10% → v2**

Apply the update:

```bash
kubectl apply -f virtualService.yaml
```

Test repeatedly:

```bash
kubectl exec -ti -n test test -- curl helloworld.default.svc.cluster.local:5000/hello
```

Expected distribution:

* v1 responses ≈ 90%
* v2 responses ≈ 10%

Example:

```
Hello version: v1...
Hello version: v1...
Hello version: v1...
Hello version: v2...
```

---

## Conclusion

**Destination Rules** and **Virtual Services** are powerful Istio features that enable fine-grained traffic management, allowing you to control traffic flow between different service versions.


