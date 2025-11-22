kubectl get ns --show-labels

kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/bookinfo/platform/kube/bookinfo.yaml

kubectl get pods

NAME                              READY   STATUS    RESTARTS   AGE
details-v1-54ffb59669-f5kmt       2/2     Running   0          11m
productpage-v1-6c58956fd9-whkdn   2/2     Running   0          11m
ratings-v1-7d7546bf89-tgz68       2/2     Running   0          11m
reviews-v1-6c7fd84f89-qrxjv       2/2     Running   0          11m
reviews-v2-57bb9fdcdf-gcjxn       2/2     Running   0          11m
reviews-v3-548fc5d9c7-htc5p       2/2     Running   0          11m

kubectl create ns test --dry-run=client -o yaml > test_ns.yaml

kubectl apply -f test_ns.yaml

k run test --image=nginx -n test --dry-run=client -o yaml
 > test_pod.yaml

kubectl apply -f test_pod.yaml

kubectl get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.108.33.2     <none>        9080/TCP   16m
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    37m
productpage   ClusterIP   10.111.102.31   <none>        9080/TCP   16m
ratings       ClusterIP   10.102.207.34   <none>        9080/TCP   16m
reviews       ClusterIP   10.104.29.226   <none>        9080/TCP   16m

kubectl exec -ti -n test test -- /bin/bash

root@test:/#curl --head productpage.default.svc.cluster.local:9080
    <!-- #Note: This works because there is no strict policy at the moment which means that all traffic can freely flow in and out of the Istio Service Mesh. -->

kubectl apply -f peerAuthentication.yaml

kubectl get pa
NAME      MODE     AGE
default   STRICT   70s

kubectl exec -ti -n test test -- /bin/bash
root@test:/# curl --head productpage.default.svc.cluster.local:9080
curl: (56) Recv failure: Connection reset by peer


kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   31m   istio-injection=enabled,kubernetes.io/metadata.name=default
...
test              Active   18m   kubernetes.io/metadata.name=test


kubectl label namespaces test istio-injection=enabled
namespace/test labeled

kubectl delete -f test_pod.yaml

kubectl apply -f test_pod.yaml

kubectl exec -ti -n test test -- /bin/bash

root@test:/#  curl --head productpage.default.svc.cluster.local:9080
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 1683
server: envoy
date: Tue, 15 Apr 2025 15:26:09 GMT
x-envoy-upstream-service-time: 19

<!-- #It should work now because both workloads in both namespaces are now communicating through the Sidecar Proxy. -->



<!-- There may be situations where you need to override the Sidecar’s default behavior by restricting outgoing traffic for a specific namespace or workload. You can do this by creating a Sidecar configuration to override any namespace’s default Sidecar behavior. -->

kubectl apply -f sidecar_default_namespace.yaml
sidecar.networking.istio.io/default created

<!-- The Sidecar configuration restricts outgoing traffic to only the test and istio-system namespaces. Traffic to the default namespace (where the Bookinfo App resides) should now be blocked.

Test the Bookinfo Application from the test pod again. It should fail with an error: Empty reply from server. -->

kubectl exec -ti -n test test -- /bin/bash
root@test:/# curl --head productpage.default.svc.cluster.local:9080
curl: (52) Empty reply from server

<!-- Adjust the Sidecar configuration to allow outgoing traffic to the default namespace and reapply it.

Update the YAML to include: -->
- hosts:
  - "./*"
  - "default/*"
  - "istio-system/*" -->


kubectl apply -f sidecar_default_namespace.yaml
sidecar.networking.istio.io/default configured

<!-- Verify that the Bookinfo Application is now accessible again from the test pod after updating the Sidecar configuration. -->

kubectl exec -ti -n test test -- /bin/bash
root@test:/# curl --head productpage.default.svc.cluster.local:9080
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 1683
server: envoy
date: Tue, 15 Apr 2025 18:32:46 GMT
x-envoy-upstream-service-time: 8

<!-- Modify the Sidecar configuration to apply only to pods with the label run: test in the test namespace. Reapply the configuration. -->

<!-- Add the following under spec :
workloadSelector:
     labels:
      run: test -->

kubectl apply -f sidecar_default_namespace.yaml
sidecar.networking.istio.io/default configured

<!-- Now, check access to the Bookinfo Application from the test pod again.

It should now fail due to the workloadSelector restriction. -->

kubectl exec -ti -n test test -- /bin/bash
root@test:/# curl --head productpage.default.svc.cluster.local:9080
curl: (52) Empty reply from server


<!-- Create a new pod named nginx in the test namespace using the nginx image.

Verify that the new nginx pod can access the Bookinfo Application successfully.


The new nginx pod can access the Bookinfo Application because the Sidecar configuration's workloadSelector only applies to pods with the label run: test. The nginx pod has a different label and thus isn't affected by the restrictions. -->

kubectl run nginx --image=nginx -n test --dry-run=client -o yaml > nginx_pod.yaml

kubectl apply -f nginx_pod.yaml

kubectl exec -ti -n test nginx -- /bin/bash
root@nginx:/# curl --head productpage.default.svc.cluster.local:9080
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 1683
server: envoy
date: Tue, 15 Apr 2025 18:57:55 GMT
x-envoy-upstream-service-time: 4