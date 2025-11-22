# Destination Rules

@Welcome to the Destination Rules Lab. By the conclusion of this lab, you will acquire the skills needed to configure Destination Rules within the Istio Service Mesh.

kubectl get ns --show-labels
NAME              STATUS   AGE     LABELS
default           Active   34m     istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   2m52s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   34m     kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   34m     kubernetes.io/metadata.name=kube-public
kube-system       Active   34m     kubernetes.io/metadata.name=kube-system

kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/helloworld/helloworld.yaml
service/helloworld created
deployment.apps/helloworld-v1 created
deployment.apps/helloworld-v2 created

k get deployments.apps 
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-v1   1/1     1            1           21s
helloworld-v2   1/1     1            1           21s

k get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.107.100.199   <none>        5000/TCP   34s
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    36m

Please confirm that both Hello World pods are showing a status of 2/2 containers running, which indicates successful Istio sidecar injection.

kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
helloworld-v1-5787f49bd8-bhwp7   2/2     Running   0          67s
helloworld-v2-6746879bdd-5nfpm   2/2     Running   0          67s

kubectl create ns test --dry-run=client -o yaml > test_ns.yaml
kubectl apply -f test_ns.yaml

kubectl run test --image=nginx -n test --dry-run=client -o yaml > test_pod.yaml
kubectl apply -f test_pod.yaml

k get ns test --show-labels 
NAME   STATUS   AGE   LABELS
test   Active   72s   istio-injection=enabled,kubernetes.io/metadata.name=test

k get po -n test
NAME   READY   STATUS    RESTARTS   AGE
test   2/2     Running   0          11s


Please verify the services associated with the Hello World App by executing the command kubectl get svc.

You should observe a service named helloworld that is listening on port 5000. Execute the following command to verify:

kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.107.100.199   <none>        5000/TCP   4m37s
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    40m

Please verify the accessibility of the Hello World service from the test pod by executing the following command:

kubectl exec -ti -n test test -- curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v2, instance: helloworld-v2-6746879bdd-5nfpm

kubectl exec -ti -n test test -- curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v1, instance: helloworld-v1-5787f49bd8-bhwp7

kubectl get pods --show-labels
NAME                             READY   STATUS    RESTARTS   AGE     LABELS
helloworld-v1-5787f49bd8-bhwp7   2/2     Running   0          7m10s   app=helloworld,pod-template-hash=5787f49bd8,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=helloworld,service.istio.io/canonical-revision=v1,version=v1
helloworld-v2-6746879bdd-5nfpm   2/2     Running   0          7m10s   app=helloworld,pod-template-hash=6746879bdd,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=helloworld,service.istio.io/canonical-revision=v2,version=v2


kubectl get svc helloworld -o yaml > helloWorldSvc.yaml

Please observe that the Hello World Service is routing traffic to pods labeled with app: helloworld. While the service itself has a service: helloworld label, this label is not part of the service's selector and does not influence routing. Only the selector field determines which pods receive traffic.

kubectl apply -f destinationRules.yaml 
destinationrule.networking.istio.io/hello-world-ds created

kubectl get destinationrules
NAME             HOST         AGE
hello-world-ds   helloworld   59s

kubectl apply -f virtualService.yaml 
virtualservice.networking.istio.io/hello-world-vs created

kubectl get vs
NAME             GATEWAYS   HOSTS            AGE
hello-world-vs              ["helloworld"]   5s

To test traffic splitting, execute the curl command multiple times from the test pod. You should observe responses alternating between the v1 and v2 versions.

root@test:/# 
root@test:/# curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v1, instance: helloworld-v1-5787f49bd8-bhwp7
root@test:/# curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v2, instance: helloworld-v2-6746879bdd-5nfpm
root@test:/# curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v1, instance: helloworld-v1-5787f49bd8-bhwp7
root@test:/# curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v1, instance: helloworld-v1-5787f49bd8-bhwp7
root@test:/# curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v2, instance: helloworld-v2-6746879bdd-5nfpm
root@test:/# curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v1, instance: helloworld-v1-5787f49bd8-bhwp7
root@test:/# curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v2, instance: helloworld-v2-6746879bdd-5nfpm

