In this Istio Ambient – Traffic Management lab, you will:

Observe the reasons behind the initial failure of a 95/5 Layer 7 split in Ambient.
Enable a Waypoint and transition to the Gateway API (HTTPRoute) for proper Layer 7 splitting.
Practice Layer 7 fault injection (delay and abort) on httpbin.

There should be two yaml files located in the current directory that we can use for this lab, one called helloworld.yaml and another called httpbin.yaml.

Deploy the hello world app inside the hello namespace by running:

kubectl apply -f helloworld.yaml -n hello

Once deployed, inspect the workloads by running:

kubectl get pods -n hello 
kubectl get svc -n hello

kubectl apply -f helloworld.yaml -n hello
service/helloworld created
deployment.apps/helloworld-v1 created
deployment.apps/helloworld-v2 created

kubectl get pods -n hello 
kubectl get svc -n hello
NAME                             READY   STATUS    RESTARTS   AGE
helloworld-v1-5787f49bd8-ln7qr   1/1     Running   0          8s
helloworld-v2-6746879bdd-p8jnq   1/1     Running   0          8s
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.109.47.128   <none>        5000/TCP   8s


Create a DestinationRule named hello-world-dr in the hello namespace.

This rule should target the helloworld service and include subsets for both v1 and v2, corresponding to the version labels on the pods: version: v1 and version: v2.

k apply -f hello-dr.yaml 
destinationrule.networking.istio.io/hello-world-dr created

root@controlplane ~ ➜  kubectl get destinationrules
No resources found in default namespace.

root@controlplane ~ ➜  kubectl get destinationrules -n hello
NAME             HOST         AGE
hello-world-dr   helloworld   16s

Create a VirtualService named hello-world-vs in the hello namespace to attempt a 95/5 traffic split between v1 and v2.

This VirtualService should route 95% of traffic to the v1 subset and 5% to the v2 subset.

k apply -f hellovs.yaml 
virtualservice.networking.istio.io/hello-world-vs created

root@controlplane ~ ➜  kubectl get virtualservice -n hello
NAME             GATEWAYS   HOSTS            AGE
hello-world-vs              ["helloworld"]   10s


Test the traffic distribution by running multiple curl requests from the test pod:

for i in {1..10}; do kubectl exec curl -n test -- curl -s helloworld.hello.svc.cluster.local:5000/hello; done
Hello version: v2, instance: helloworld-v2-6746879bdd-p8jnq
Hello version: v1, instance: helloworld-v1-5787f49bd8-ln7qr
Hello version: v1, instance: helloworld-v1-5787f49bd8-ln7qr
Hello version: v2, instance: helloworld-v2-6746879bdd-p8jnq
Hello version: v2, instance: helloworld-v2-6746879bdd-p8jnq
Hello version: v1, instance: helloworld-v1-5787f49bd8-ln7qr
Hello version: v1, instance: helloworld-v1-5787f49bd8-ln7qr
Hello version: v1, instance: helloworld-v1-5787f49bd8-ln7qr
Hello version: v1, instance: helloworld-v1-5787f49bd8-ln7qr
Hello version: v2, instance: helloworld-v2-6746879bdd-p8jnq




What do you notice? Doesn’t seem correct right? 95% of all traffic should be going to v1 but instead it’s alternating 50/50. Why is that?
Inspect the hello namespace by running:

root@controlplane ~ ➜  kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   15m   kubernetes.io/metadata.name=default
hello             Active   13m   kubernetes.io/metadata.name=hello
httpbin           Active   13m   kubernetes.io/metadata.name=httpbin
istio-system      Active   13m   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   15m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   15m   kubernetes.io/metadata.name=kube-public
kube-system       Active   15m   kubernetes.io/metadata.name=kube-system
test              Active   13m   istio.io/dataplane-mode=ambient,istio.io/use-waypoint=waypoint,kubernetes.io/metadata.name=test

Is there something missing? Unfortunately, in order to use split traffic capabilities we need to use the Waypoint Proxy.

This demonstrates a key limitation of Ambient mode without Waypoint proxies - advanced Layer 7 features like weighted routing require a Waypoint to be enabled.


Enable Ambient mode and Waypoint proxy for the hello namespace, then delete the old VirtualService and DestinationRule to prepare for Gateway API usage.


Now the hello namespace will use Ambient Mode for all Layer 4 traffic and the Waypoint Proxy for all Layer 7 traffic.

kubectl label ns hello istio.io/dataplane-mode=ambient istio.io/use-waypoint=waypoint --overwrite
namespace/hello labeled

root@controlplane ~ ➜  istioctl waypoint apply -n hello
✅ waypoint hello/waypoint applied

root@controlplane ~ ➜  kubectl get deploy waypoint -n hello
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
waypoint   1/1     1            1           9s

root@controlplane ~ ➜  kubectl delete virtualservice hello-world-vs -n hello --ignore-not-found
kubectl delete destinationrule hello-world-dr -n hello --ignore-not-found
virtualservice.networking.istio.io "hello-world-vs" deleted
destinationrule.networking.istio.io "hello-world-dr" deleted

Create an HTTPRoute named hello-http-split-traffic in the hello namespace to achieve the 95/5 traffic split using Gateway API.


The HTTPRoute should reference the main helloworld service as a parentRef and distribute traffic to separate helloworld-v1 and helloworld-v2 services with 95/5 weights.

kubectl get crd httproutes.gateway.networking.k8s.io -o name
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io

root@controlplane ~ ➜  kubectl apply -n hello -f helloworld.yaml
service/helloworld unchanged
deployment.apps/helloworld-v1 unchanged
deployment.apps/helloworld-v2 unchanged

root@controlplane ~ ➜  kubectl get svc -n hello
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)               AGE
helloworld   ClusterIP   10.109.47.128   <none>        5000/TCP              12m
waypoint     ClusterIP   10.100.95.103   <none>        15021/TCP,15008/TCP   2m21s

root@controlplane ~ ➜  cat > hello-httproute-split-traffic.yaml <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-http-split-traffic
  namespace: hello
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: helloworld
    port: 5000
  rules:
  - backendRefs:
    - name: helloworld-v1
      port: 5000
      weight: 95
    - name: helloworld-v2
      port: 5000
      weight: 5
EOF
kubectl apply -f hello-httproute-split-traffic.yaml
kubectl get httproutes -n hello
httproute.gateway.networking.k8s.io/hello-http-split-traffic created
NAME                       HOSTNAMES   AGE
hello-http-split-traffic               0s

