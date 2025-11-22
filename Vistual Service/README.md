# Virtual Service

kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   45m   istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   80s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   45m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   45m   kubernetes.io/metadata.name=kube-public
kube-system       Active   45m   kubernetes.io/metadata.name=kube-system

kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml
serviceaccount/httpbin created
service/httpbin created
deployment.apps/httpbin created

kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-686d6fc899-vlcsp   2/2     Running   0          28s

kubectl create ns test --dry-run=client -o yaml > test_ns.yaml
kubectl apply -f test_ns.yaml

kubectl run test --image=nginx -n test --dry-run=client -o yaml > test_pod.yaml
kubectl apply -f test_pod.yaml

Retrieve the services associated with the HTTP Bin App

kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
httpbin      ClusterIP   10.107.246.109   <none>        8000/TCP   3m31s
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    54m


You should observe a service named httpbin that listens on port 8000.
Next, confirm that the HTTP Bin service is accessible from the test pod.

kubectl exec -ti -n test test -- curl --head httpbin.default.svc.cluster.local:8000
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-security-policy: default-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' camo.githubusercontent.com
content-type: text/html; charset=utf-8
date: Sat, 22 Nov 2025 11:22:16 GMT
x-envoy-upstream-service-time: 1
server: istio-envoy
x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*
transfer-encoding: chunked



create a Virtual Service named httpbin in the default namespace to route all HTTP traffic (specifically, URIs that begin with '/') to the service httpbin.default.svc.cluster.local on port 8000.

kubectl apply -f v virtualService.yaml

kubectl get vs
NAME      GATEWAYS   HOSTS         AGE
httpbin              ["httpbin"]   51s


The current configuration of the Virtual Service mirrors the behavior of the HTTP Bin service. At this stage, no changes will be observed.


kubectl exec -ti -n test test -- curl --head httpbin.default.svc.cluster.local:8000
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-security-policy: default-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' camo.githubusercontent.com
content-type: text/html; charset=utf-8
date: Sat, 22 Nov 2025 11:28:27 GMT
x-envoy-upstream-service-time: 0
server: istio-envoy
x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*
transfer-encoding: chunked

Please modify the virtualService.yaml file to change the destination port to 9000, and then reapply the configuration to break the Virtual Service.

kubectl apply -f virtualService.yaml
virtualservice.networking.istio.io/httpbin configured

kubectl exec -ti -n test test -- curl --head httpbin.default.svc.cluster.local:8000
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-security-policy: default-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' camo.githubusercontent.com
content-type: text/html; charset=utf-8
date: Sat, 22 Nov 2025 11:33:22 GMT
x-envoy-upstream-service-time: 0
server: istio-envoy
x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*
transfer-encoding: chunked

This indicates that the test namespace does not have Istio injection enabled, resulting in the Virtual Service being ignored.

You can verify the namespace labels by executing the following command:

kubectl get ns test --show-labels
NAME   STATUS   AGE   LABELS
test   Active   13m   kubernetes.io/metadata.name=test


Please enable Istio injection in the test namespace. After completing this step, proceed to delete and then recreate the test pod.
Edit test_ns.yaml for lable.

kubectl delete -f test_pod.yaml
kubectl apply -f test_pod.yaml

kubectl get po
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-686d6fc899-vlcsp   2/2     Running   0          17m