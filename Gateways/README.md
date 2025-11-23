By the conclusion of this lab, you will gain the skills necessary to configure Ingress Gateways within the Istio Service Mesh.

Please be aware that the default namespace has Istio injection enabled. You can verify this by executing the following command:


kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   24m   istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   66s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   24m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   24m   kubernetes.io/metadata.name=kube-public
kube-system       Active   24m   kubernetes.io/metadata.name=kube-system


kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/bookinfo/platform/kube/bookinfo.yaml

service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created

kubectl get svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.101.176.103   <none>        9080/TCP   24s
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    16m
productpage   ClusterIP   10.100.87.65     <none>        9080/TCP   21s
ratings       ClusterIP   10.101.174.213   <none>        9080/TCP   24s
reviews       ClusterIP   10.107.41.56     <none>        9080/TCP   23s

kubectl get sa
NAME                   SECRETS   AGE
bookinfo-details       0         27s
bookinfo-productpage   0         24s
bookinfo-ratings       0         27s
bookinfo-reviews       0         26s
default                0         16m

kubectl get deploy
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
details-v1       1/1     1            1           38s
productpage-v1   0/1     1            0           35s
ratings-v1       1/1     1            1           38s
reviews-v1       0/1     1            0           37s
reviews-v2       1/1     1            1           36s
reviews-v3       0/1     1            0           36s

kubectl get po
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-54ffb59669-5pj8v       2/2     Running   0          67s
productpage-v1-6c58956fd9-7gs2m   2/2     Running   0          64s
ratings-v1-7d7546bf89-9fv66       2/2     Running   0          66s
reviews-v1-6c7fd84f89-rc242       2/2     Running   0          65s
reviews-v2-57bb9fdcdf-6lq7n       2/2     Running   0          65s
reviews-v3-548fc5d9c7-m8cwl       2/2     Running   0          65s

To expose the Bookinfo application publicly, we first need to create a Virtual Service in the istio-system namespace where the ingress gateway lives.

The Virtual Service will define routing rules for incoming traffic.

kubectl apply -f virtualService.yaml
virtualservice.networking.istio.io/book-info-vs created

Before creating the Gateway, please identify the appropriate label for the ingress gateway pod.
Run the following command:

kubectl describe pod -n istio-system istio-ingress | grep Labels -A 20

Labels:           app=istio-ingressgateway
                  app.kubernetes.io/instance=istio
                  app.kubernetes.io/managed-by=Helm
                  app.kubernetes.io/name=istio-ingressgateway
                  app.kubernetes.io/part-of=istio
                  app.kubernetes.io/version=1.26.0
                  chart=gateways
                  helm.sh/chart=istio-ingress-1.26.0
                  heritage=Tiller
                  install.operator.istio.io/owning-resource=unknown
                  istio=ingressgateway
                  istio.io/dataplane-mode=none
                  istio.io/rev=default
                  operator.istio.io/component=IngressGateways
                  pod-template-hash=6cd9bc7f5b
                  release=istio
                  service.istio.io/canonical-name=istio-ingressgateway
                  service.istio.io/canonical-revision=latest
                  sidecar.istio.io/inject=false

With the Virtual Service prepared, you can now proceed to create the Gateway resource.

Create a Gateway resource named istio-gateway in the istio-system namespace to expose HTTP traffic on port 80 for the host book.info.com.


In your Gateway YAML configuration, the hosts field represents the DNS record that users will utilize to access this workload.

Please configure the hosts field to book.info.com. It is important to note that in a production environment, this should correspond to the actual DNS record of the website, which points to the IP address of the Load Balancer.

kubectl apply -f gateway.yaml
gateway.networking.istio.io/istio-gateway created

kubectl get gateways -A
NAMESPACE      NAME            AGE
istio-system   istio-gateway   8s

Please take note of the NodePort IP that has been provisioned. You can verify this information by executing the command: kubectl get svc -n istio-system.

Please execute the following command to retrieve the services within the "istio-system" namespace:

kubectl get svc -n istio-system
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP   10.101.133.25   <none>        80/TCP,443/TCP                                                               12m
istio-ingressgateway   NodePort    10.98.54.69     <none>        15021:30994/TCP,80:32177/TCP,443:31325/TCP,31400:31375/TCP,15443:31323/TCP   12m
istiod                 ClusterIP   10.110.217.46   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP                                        12m

This command will display the list of services, including their name, type, Cluster-IP, external IP, ports, and age.

Please test your access to the Bookinfo application via the ingress gateway by executing the following curl command, while substituting <NodePort IP> with the IP address obtained in the previous step.

curl --head --header "Host: book.info.com" http://10.98.54.69
HTTP/1.1 404 Not Found
date: Sun, 23 Nov 2025 05:39:58 GMT
server: istio-envoy
transfer-encoding: chunked

The command does not appear to be functioning as expected. Please consider the possible reasons for this issue, and review the subsequent questions for further clarification.

Let’s have a look at the Virtual Service again.

The issue at hand is that the Virtual Service is set to intercept traffic directed to the productpage svc. However, when accessing from outside, users do not input productpage; instead, they utilize the DNS Record. Additionally, we have neglected to include the gateways parameter, which allows us to associate the Gateway we created with the Virtual Service.

Update the Virtual Service configuration to include the external hostname and gateway reference, then reapply it.

kubectl apply -f virtualService.yaml
virtualservice.networking.istio.io/book-info-vs configured

Please test the reachability of the Bookinfo application once again.

Execute the following command:

curl --head --header "Host: book.info.com" http://10.98.54.69
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 1683
server: istio-envoy
date: Sun, 23 Nov 2025 05:45:44 GMT
x-envoy-upstream-service-time: 21

