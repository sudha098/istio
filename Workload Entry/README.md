
# 📘 Istio WorkloadEntry & ServiceEntry Lab

**Onboarding External (Non-Kubernetes) Workloads into the Istio Service Mesh**

This lab walks through how to register an external workload (NGINX running directly on the host) into the Istio service mesh using **WorkloadEntry** and **ServiceEntry** resources.
By the end, you will:

* ✔ Install Istio with **REGISTRY_ONLY** outbound mode
* ✔ Register a host-based workload into the mesh
* ✔ Expose it through a ServiceEntry
* ✔ Access it safely from inside the mesh
* ✔ Demonstrate that Istio load balances across both external and internal workloads

---

# 📑 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Verify Kubernetes & Istio Installation](#verify-kubernetes--istio-installation)
3. [Install Istio with REGISTRY_ONLY](#install-istio-with-registry_only)
4. [Label Default Namespace for Injection](#label-default-namespace-for-injection)
5. [Verify Host-Level NGINX](#verify-host-level-nginx)
6. [Test Mesh Outbound Restrictions](#test-mesh-outbound-restrictions)
7. [Create WorkloadEntry](#create-workloadentry)
8. [Create ServiceEntry](#create-serviceentry)
9. [Access External Service via Mesh](#access-external-service-via-mesh)
10. [Add an In-Cluster NGINX (Load Balancing Demo)](#add-an-in-cluster-nginx-load-balancing-demo)
11. [Final Verification](#final-verification)
12. [Conclusion](#conclusion)

---

# 🧰 Prerequisites

* A running Kubernetes cluster (kind, kubeadm, Minikube, etc.)
* Istioctl installed (1.26 or newer recommended)
* Host has NGINX installed (`curl localhost` should show default page)

---

# 1️⃣ Verify Kubernetes & Istio Installation

```bash
kubectl get pods -A
```

You should see default Kubernetes system pods running.

---

# 2️⃣ Install Istio With `REGISTRY_ONLY`

This outbound mode ensures that **all external traffic must be explicitly registered**.

```bash
istioctl install \
  --set profile=demo \
  --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
```

Verify:

```bash
kubectl get configmap istio -n istio-system -o yaml | grep outbound -A 2
```

Expected:

```
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
```

---

# 3️⃣ Label Default Namespace for Sidecar Injection

```bash
kubectl label ns default istio-injection=enabled
```

---

# 4️⃣ Verify Host-Level NGINX

Test that NGINX on the host is running:

```bash
curl http://localhost
```

Next, obtain the **weave** network interface IP:

```bash
ip a
```

Example:

```
5: weave:
    inet 10.50.0.3/16
```

Test from host:

```bash
curl http://10.50.0.3
```

---

# 5️⃣ Test Mesh Outbound Restrictions

Create test pod:

```bash
kubectl run test --image=nginx
```

Try to curl the host NGINX **from inside the mesh**:

```bash
kubectl exec -ti test -- curl --head http://10.50.0.3
```

Expected:

```
HTTP/1.1 502 Bad Gateway
```

✔ This confirms **REGISTRY_ONLY is blocking unknown destinations**.

---

# 6️⃣ Create WorkloadEntry

`workload_entry.yaml`:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadEntry
metadata:
  name: external-app-we
spec:
  address: 10.50.0.3
  labels:
    app: external
```

Apply:

```bash
kubectl apply -f workload_entry.yaml
kubectl get workloadentry
```

Expected:

```
external-app-we   10.50.0.3
```

---

# 7️⃣ Create ServiceEntry

This makes `app.internal.com` a **registered service** inside the mesh.

`service_entry.yaml`:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-app-se
spec:
  hosts:
    - app.internal.com
  resolution: STATIC
  ports:
    - number: 80
      name: http
      protocol: HTTP
  workloadSelector:
    labels:
      app: external
```

Apply:

```bash
kubectl apply -f service_entry.yaml
kubectl get serviceentry
```

---

# 8️⃣ Access External Service Through the Mesh (Now Works)

```bash
kubectl exec -ti test -- curl http://app.internal.com
```

You should see the host NGINX welcome page.

⚠ Accessing the raw IP still fails:

```bash
kubectl exec -ti test -- curl http://10.50.0.3
```

Expected:

```
HTTP/1.1 502 Bad Gateway
```

✔ Mesh behavior is correct.

---

# 9️⃣ Add an In-Cluster NGINX (Load Balancing Demo)

Create pod with same label:

```bash
kubectl run nginx --image=nginx --labels="app=external"
```

Modify its index page:

```bash
kubectl exec -ti nginx -- bash
cd /usr/share/nginx/html
echo "This is an Nginx Pod" > index.html
curl localhost
exit
```

---

# 🔟 Final Verification

Call the service again:

```bash
kubectl exec -ti test -- curl http://app.internal.com
```

You should see **two alternating responses**:

### Response from external host NGINX:

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

### Response from in-cluster NGINX pod:

```
This is an Nginx Pod
```

🎉 Congratulations — Istio is load balancing between:

* The **WorkloadEntry** (external host workload)
* The **in-cluster pod** (Kubernetes workload)

All under a single service name: **app.internal.com**

---

# 🏁 Conclusion

In this lab you successfully:

* Installed Istio with **strict outbound policy**
* Onboarded an external workload using **WorkloadEntry**
* Registered a new DNS service using **ServiceEntry**
* Verified that unregistered traffic is blocked by REGISTRY_ONLY
* Demonstrated Istio load balancing between:

  * a **host-level process**, and
  * a **Kubernetes pod**

This enables powerful patterns such as:

* Migrating legacy VMs into the service mesh
* Hybrid mesh workloads (VMs + Pods)
* Traffic splitting
* Traffic shadowing
* Policy enforcement for external services

---
