# ✅ Updating the VirtualService to Use Retry Policy

Replace the previous timeout block with the following **retry configuration**:

### **virtualservice.yaml**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-vs
spec:
  hosts:
    - httpbin
  http:
    - retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: "5xx"
      route:
        - destination:
            host: httpbin
            port:
              number: 8000
```

Apply it:

```bash
kubectl apply -f virtualservice.yaml
```

Expected:

```
virtualservice.networking.istio.io/httpbin-vs created
```

This retry policy means:

* Istio retries **up to 3 times**
* Each attempt has a max duration of **1 second**
* Retries occur only on **5xx responses**
* All retries redirect to the same service (`httpbin:8000`)

---

# ✅ Testing the Retry Mechanism

Test the `/status/500` endpoint, which **always returns HTTP 500**.

```bash
kubectl exec -ti test -- curl -v http://httpbin.default.svc:8000/status/500
```

Example response:

```
< HTTP/1.1 500 Internal Server Error
< server: envoy
```

Even after retries, you still get a 500 — this is expected since the endpoint always fails.

---

# ✅ Verifying Retry Attempts in the Istio Proxy Logs

To confirm retries, check the `istio-proxy` container logs:

```bash
kubectl logs httpbin-686d6fc899-8hx55 -c istio-proxy
```

Look for **4 GET requests** for the same request ID:

* 1 original request
* 3 retry attempts

Example excerpt (simplified):

```
"GET /status/500 HTTP/1.1" 500 -
"GET /status/500 HTTP/1.1" 500 -
"GET /status/500 HTTP/1.1" 500 -
"GET /status/500 HTTP/1.1" 500 -
```

This confirms Istio retried 3 times as configured.

---

# ✅ Adjusting Retry Attempts (Set to 1)

Update the VirtualService:

```yaml
retries:
  attempts: 1
  perTryTimeout: 1s
  retryOn: "5xx"
```

Apply:

```bash
kubectl apply -f virtualservice.yaml
```

Test again:

```bash
kubectl exec -ti test -- curl -v http://httpbin.default.svc:8000/status/500
```

You still receive:

```
HTTP/1.1 500 Internal Server Error
```

Check logs:

```bash
kubectl logs httpbin-686d6fc899-8hx55 -c istio-proxy
```

You should now see **only 2 log entries**:

* 1 original request
* 1 retry

Confirmed example:

```
"GET /status/500 HTTP/1.1" 500 -
"GET /status/500 HTTP/1.1" 500 -
```

This demonstrates that the retry behavior changes correctly as you adjust the configuration.

---

# 🎉 Retry Policy Lab Complete

You have now successfully configured and validated:

✔️ Retry attempts
✔️ Per-try timeout
✔️ RetryOn rules
✔️ Log verification inside Envoy
✔️ Behavior changes when retry attempts are modified

