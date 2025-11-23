By the end of this lab, you will be able to configure timeouts and implement retry policies for Fault Injection in the Istio Service Mesh.

Verify that your Kubernetes environment is correctly set up by checking that the default namespace is Istio enabled.
Run: kubectl get ns --show-labels and confirm the 'istio-injection=enabled' label.

You should the see the output as:

kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   20m   istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   79s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   20m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   20m   kubernetes.io/metadata.name=kube-public
kube-system       Active   20m   kubernetes.io/metadata.name=kube-system

Deploy the HTTP Bin Application by running:

kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml
serviceaccount/httpbin created
service/httpbin created
deployment.apps/httpbin created

kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-686d6fc899-b9dht   2/2     Running   0          56s

Run a test pod with nginx by executing:

kubectl run test --image=nginx
pod/test created

k get po
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-686d6fc899-8hx55   2/2     Running   0          26s
test                       2/2     Running   0          17s

Retrieve the services for the HTTP Bin App by executing the command kubectl get svc. You should observe a service named "httpbin" that is configured to listen on port 8000.

To test the HTTP Bin endpoint, utilize the /delayfunctionality. The httpbin workload features a /delay/<seconds> endpoint available for this purpose.
Run the following command:
kubectl exec -ti test -- curl --head http://httpbin.default.svc:8000/delay/5
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-type: application/json; charset=utf-8
server-timing: initial_delay;dur=5000.00;desc="initial delay"
date: Sun, 23 Nov 2025 12:54:13 GMT
x-envoy-upstream-service-time: 5006
server: envoy
transfer-encoding: chunked

A response of HTTP/1.1 200 OK is anticipated.

You can test httpbin and add 5 or 10 seconds if you want to see it in action.

Create a Virtual Service httpbin-vs to enforce a timeout on the HTTP Bin application. Your VirtualService should:

Target host httpbin
Set timeout: 2s in the HTTP block
Route traffic to port 8000
Apply the Virtual Service configuration and verify its creation by executing kubectl get vs.


The Virtual Service states that all traffic going to httpbin will time out after 2 seconds if the app doesn’t respond.

kubectl apply -f virtualservice.yaml 
virtualservice.networking.istio.io/httpbin-vs created

root@controlplane ~ ➜  kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
httpbin      ClusterIP   10.109.240.230   <none>        8000/TCP   3m49s
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    28m

Test the Virtual Service timeout behavior:

Run this command; it should return HTTP/1.1 200 OK.

kubectl exec -ti test -- curl --head http://httpbin.default.svc:8000/delay/1
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-type: application/json; charset=utf-8
server-timing: initial_delay;dur=1000.00;desc="initial delay"
date: Sun, 23 Nov 2025 12:57:55 GMT
x-envoy-upstream-service-time: 1002
server: envoy
transfer-encoding: chunked

Then, run this command, which should return a 504 Gateway Timeout.
kubectl exec -ti test -- curl --head http://httpbin.default.svc:8000/delay/5
HTTP/1.1 504 Gateway Timeout
content-length: 24
content-type: text/plain
date: Sun, 23 Nov 2025 12:58:03 GMT
server: envoy
