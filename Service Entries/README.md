By the end of this lab, you will have the skills to configure Service Entries and Egress Gateways within the Istio Service Mesh.

To verify that Kubernetes is operational, please execute the command kubectl get pods -A. Confirm that the default Kubernetes workloads are currently active.

kubectl get pods -A
NAMESPACE     NAME                                   READY   STATUS    RESTARTS      AGE
kube-system   coredns-668d6bf9bc-6xc92               1/1     Running   0             17m
kube-system   coredns-668d6bf9bc-b48vs               1/1     Running   0             17m
kube-system   etcd-controlplane                      1/1     Running   0             18m
kube-system   kube-apiserver-controlplane            1/1     Running   0             18m
kube-system   kube-controller-manager-controlplane   1/1     Running   0             18m
kube-system   kube-proxy-7mrhv                       1/1     Running   0             17m
kube-system   kube-scheduler-controlplane            1/1     Running   0             18m
kube-system   weave-net-g9854                        2/2     Running   1 (17m ago)   17m

Download and install Istio version 1.26.0 using the command:

curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.26.0 sh -
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   101  100   101    0     0    678      0 --:--:-- --:--:-- --:--:--   682
100  5124  100  5124    0     0  17857      0 --:--:-- --:--:-- --:--:-- 42347

Downloading istio-1.26.0 from https://github.com/istio/istio/releases/download/1.26.0/istio-1.26.0-linux-amd64.tar.gz ...

Istio 1.26.0 download complete!

The Istio release archive has been downloaded to the istio-1.26.0 directory.

To configure the istioctl client tool for your workstation,
add the /root/istio-1.26.0/bin directory to your environment path variable with:
         export PATH="$PATH:/root/istio-1.26.0/bin"

Begin the Istio pre-installation check by running:
         istioctl x precheck 

Try Istio in ambient mode
        https://istio.io/latest/docs/ambient/getting-started/
Try Istio in sidecar mode
        https://istio.io/latest/docs/setup/getting-started/
Install guides for ambient mode
        https://istio.io/latest/docs/ambient/install/
Install guides for sidecar mode
        https://istio.io/latest/docs/setup/install/

Need more information? Visit https://istio.io/latest/docs/ 


Copy the Utility to /usr/local/bin/

cp /root/istio-1.26.0/bin/istioctl /usr/local/bin/

Now, verify the installation using:

istioctl version
Istio is not present in the cluster: no running Istio pods in namespace "istio-system"
client version: 1.26.0

To demonstrate Istio's method of controlling access to external services, we need to modify the Istio installation configuration to set the outbound traffic policy to REGISTRY_ONLY.

Install Istio using the demo profile along with the required adjustment.

Task Instructions:

Execute the following command to install Istio with the demo profile and meshConfig.outboundTrafficPolicy.mode option set to REGISTRY_ONLY

istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
        |\          
        | \         
        |  \        
        |   \       
      /||    \      
     / ||     \     
    /  ||      \    
   /   ||       \   
  /    ||        \  
 /     ||         \ 
/______||__________\
____________________
  \__       _____/  
     \_____/        

This will install the Istio 1.26.0 profile "demo" into the cluster. Proceed? (y/N) y
✔ Istio core installed ⛵️                                                                                            
✔ Istiod installed 🧠                                                                                                
✔ Egress gateways installed 🛫                                                                                      .
✔ Ingress gateways installed 🛬                                                                                     
✔ Installation complete         

Validate that the configuration is successfully set using the command:

kubectl get configmap istio -n istio-system -o yaml
apiVersion: v1
data:
  mesh: |-
    accessLogFile: /dev/stdout
    defaultConfig:
      discoveryAddress: istiod.istio-system.svc:15012
    defaultProviders:
      metrics:
      - prometheus
    enablePrometheusMerge: true
    extensionProviders:
    - envoyOtelAls:
        port: 4317
        service: opentelemetry-collector.observability.svc.cluster.local
      name: otel
    - name: skywalking
      skywalking:
        port: 11800
        service: tracing.istio-system.svc.cluster.local
    - name: otel-tracing
      opentelemetry:
        port: 4317
        service: opentelemetry-collector.observability.svc.cluster.local
    - name: jaeger
      opentelemetry:
        port: 4317
        service: jaeger-collector.istio-system.svc.cluster.local
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
    rootNamespace: istio-system
    trustDomain: cluster.local
  meshNetworks: 'networks: {}'
kind: ConfigMap
metadata:
  creationTimestamp: "2025-11-23T05:58:59Z"
  labels:
    app.kubernetes.io/instance: istio
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: istiod
    app.kubernetes.io/part-of: istio
    app.kubernetes.io/version: 1.26.0
    helm.sh/chart: istiod-1.26.0
    install.operator.istio.io/owning-resource: unknown
    install.operator.istio.io/owning-resource-namespace: istio-system
    istio.io/rev: default
    operator.istio.io/component: Pilot
    operator.istio.io/managed: Reconcile
    operator.istio.io/version: 1.26.0
    release: istio
  name: istio
  namespace: istio-system
  resourceVersion: "2136"
  uid: a9872836-0ee7-4129-a079-819f244fe073

kubectl get configmap istio -n istio-system -o yaml | grep outboundTrafficPolicy -A 2
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
    rootNamespace: istio-system

You should be able to see the below in the output under mesh config:

kubectl get configmap istio -n istio-system -o yaml | grep outboundTrafficPolicy -A 2
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
    rootNamespace: istio-system

Alternatively, create an IstioOperator file and add the following inside the meshConfig section:

apiVersion: install.istio.io/v1
kind: IstioOperator
spec:
  profile: demo
  meshConfig:
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
Validate the file by running:

istioctl validate -f demo.yaml
Install Istio by running:

istioctl install -f demo.yaml -y
Confirm that Istio is successfully installed by running the command below:
   kubectl get pods -n istio-system
The expected output should indicate that all pods are in a running state, similar to below:

kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-675cdb9f4b-qngqt    1/1     Running   0          4m11s
istio-ingressgateway-6cd9bc7f5b-ngnl5   1/1     Running   0          4m11s
istiod-7f898458c5-5c9ld                 1/1     Running   0          4m21s

Create a test pod using the nginx image and verify its external access to www.wikipedia.org.

You can expect a 200 response. This is due to the fact that the default namespace is not enabled for Istio.

kubectl run test --image=nginx
pod/test created

kubectl exec -ti test -- curl --head -L http://www.wikipedia.org
HTTP/1.1 301 Moved Permanently
content-length: 0
location: https://www.wikipedia.org/
server: HAProxy
x-cache: cp2039 int
x-cache-status: int-tls
connection: close

HTTP/2 200 
date: Sat, 22 Nov 2025 18:17:43 GMT
cache-control: s-maxage=86400, must-revalidate, max-age=3600
server: ATS/9.2.11
etag: W/"1bb6c-6426794fdf540"
last-modified: Thu, 30 Oct 2025 22:15:09 GMT
content-type: text/html
age: 42431
accept-ranges: bytes
x-cache: cp2041 hit, cp2031 hit/435379
x-cache-status: hit-front
server-timing: cache;desc="hit-front", host;desc="cp2031"
strict-transport-security: max-age=106384710; includeSubDomains; preload
report-to: { "group": "wm_nel", "max_age": 604800, "endpoints": [{ "url": "https://intake-logging.wikimedia.org/v1/events?stream=w3c.reportingapi.network_error&schema_uri=/w3c/reportingapi/network_error/1.0.0" }] }
nel: { "report_to": "wm_nel", "max_age": 604800, "failure_fraction": 0.05, "success_fraction": 0.0}
set-cookie: WMF-Last-Access=23-Nov-2025;Path=/;HttpOnly;secure;Expires=Thu, 25 Dec 2025 00:00:00 GMT
set-cookie: WMF-Last-Access-Global=23-Nov-2025;Path=/;Domain=.wikipedia.org;HttpOnly;secure;Expires=Thu, 25 Dec 2025 00:00:00 GMT
x-client-ip: 35.188.139.128
set-cookie: GeoIP=US:IA:Council_Bluffs:41.26:-95.85:v4; Path=/; secure; Domain=.wikipedia.org
set-cookie: NetworkProbeLimit=0.001;Path=/;Secure;SameSite=None;Max-Age=3600
set-cookie: WMF-Uniq=QbGZtR0JzCmqiTAu-Ys4SgK0AAAAAFvdw9qUgQykvvgzngyrnQPtduk01DZyhwnu;Domain=.wikipedia.org;Path=/;HttpOnly;secure;SameSite=None;Expires=Mon, 23 Nov 2026 00:00:00 GMT
content-length: 113516
x-request-id: e413e45e-5a28-4b36-91b7-8233ab9fb27b
x-analytics: 

Label the default namespace with the label istio-injection=enabled. Subsequently, delete and recreate the test pod.

Once completed, test the connection from within the test pod to www.wikipedia.org again.


You will observe that the curl command no longer functions, and the test pod is unable to access wikipedia.com. This is due to the fact that outbound traffic is strictly configured to REGISTRY_ONLY.

kubectl label ns default istio-injection=enabled
namespace/default labeled

kubectl delete po test
pod "test" deleted

kubectl run test --image=nginx
pod/test created

kubectl exec -ti test -- curl --head -L http://www.wikipedia.org
HTTP/1.1 502 Bad Gateway
date: Sun, 23 Nov 2025 06:06:41 GMT
server: envoy
transfer-encoding: chunked

Create a ServiceEntry named wikipedia-egress to allow external HTTP and HTTPS traffic to www.wikipedia.org, and verify that the configuration is working as expected.


With the Service Entry resource created, wikipedia.org should work without any problem.

kubectl apply -f service_entry.yaml 
serviceentry.networking.istio.io/wikipedia-egress created

kubectl exec -ti test -- curl --head -L http://www.wikipedia.org
HTTP/1.1 301 Moved Permanently
content-length: 0
location: https://www.wikipedia.org/
server: envoy
x-cache: cp2039 int
x-cache-status: int-tls
x-envoy-upstream-service-time: 52
date: Sun, 23 Nov 2025 06:09:10 GMT

HTTP/2 200 
date: Sat, 22 Nov 2025 18:17:43 GMT
cache-control: s-maxage=86400, must-revalidate, max-age=3600
server: ATS/9.2.11
etag: W/"1bb6c-6426794fdf540"
last-modified: Thu, 30 Oct 2025 22:15:09 GMT
content-type: text/html
age: 42686
accept-ranges: bytes
x-cache: cp2041 hit, cp2031 hit/437875
x-cache-status: hit-front
server-timing: cache;desc="hit-front", host;desc="cp2031"
strict-transport-security: max-age=106384710; includeSubDomains; preload
report-to: { "group": "wm_nel", "max_age": 604800, "endpoints": [{ "url": "https://intake-logging.wikimedia.org/v1/events?stream=w3c.reportingapi.network_error&schema_uri=/w3c/reportingapi/network_error/1.0.0" }] }
nel: { "report_to": "wm_nel", "max_age": 604800, "failure_fraction": 0.05, "success_fraction": 0.0}
set-cookie: WMF-Last-Access=23-Nov-2025;Path=/;HttpOnly;secure;Expires=Thu, 25 Dec 2025 00:00:00 GMT
set-cookie: WMF-Last-Access-Global=23-Nov-2025;Path=/;Domain=.wikipedia.org;HttpOnly;secure;Expires=Thu, 25 Dec 2025 00:00:00 GMT
x-client-ip: 35.188.139.128
set-cookie: GeoIP=US:IA:Council_Bluffs:41.26:-95.85:v4; Path=/; secure; Domain=.wikipedia.org
set-cookie: NetworkProbeLimit=0.001;Path=/;Secure;SameSite=None;Max-Age=3600
set-cookie: WMF-Uniq=qFnGRc_qK4puCCNLEvL2ngK0AAAAAFvd0rGaayCLvba_kpQl_pabz73bM9tR16Rj;Domain=.wikipedia.org;Path=/;HttpOnly;secure;SameSite=None;Expires=Mon, 23 Nov 2026 00:00:00 GMT
content-length: 113516
x-request-id: 5c52c8ba-1d44-4d34-b36e-5c3218d88411
x-analytics: 


Please note that, after creating the ServiceEntry, while access to wikipedia.org is functioning, other external sites such as google.com remain inaccessible. This issue arises due to the REGISTRY_ONLY outbound policy that is currently in effect.

Analyze the following output from executing a kubectl exec command:

kubectl exec -ti test -- curl --head -L http://www.google.com
HTTP/1.1 502 Bad Gateway
date: Sun, 23 Nov 2025 06:10:22 GMT
server: envoy
transfer-encoding: chunked

What does the HTTP response code "502 Bad Gateway" indicate in this context?

This lab scenario involves an Ingress and Egress Gateway. You will be tasked with tailing the logs of the Egress Gateway to observe the behavior when making requests to wikipedia.org or google.com.

Instructions:

Retrieve the Egress Gateway pod name by executing the following command:

kubectl get pod -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-675cdb9f4b-qngqt    1/1     Running   0          12m
istio-ingressgateway-6cd9bc7f5b-ngnl5   1/1     Running   0          12m
istiod-7f898458c5-5c9ld                 1/1     Running   0          12m

In a separate terminal window, tail the logs by executing:

kubectl logs -f -n istio-system istio-egressgateway-675cdb9f4b-qngqt
2025-11-23T05:59:12.272578Z     info    FLAG: --concurrency="0"
2025-11-23T05:59:12.272630Z     info    FLAG: --domain="istio-system.svc.cluster.local"
2025-11-23T05:59:12.272635Z     info    FLAG: --help="false"
2025-11-23T05:59:12.272637Z     info    FLAG: --log_as_json="false"
2025-11-23T05:59:12.272638Z     info    FLAG: --log_caller=""
2025-11-23T05:59:12.272640Z     info    FLAG: --log_output_level="default:info"
2025-11-23T05:59:12.272642Z     info    FLAG: --log_stacktrace_level="default:none"
2025-11-23T05:59:12.272648Z     info    FLAG: --log_target="[stdout]"
2025-11-23T05:59:12.272650Z     info    FLAG: --meshConfig="./etc/istio/config/mesh"
2025-11-23T05:59:12.272651Z     info    FLAG: --outlierLogPath=""
2025-11-23T05:59:12.272653Z     info    FLAG: --profiling="true"
2025-11-23T05:59:12.272654Z     info    FLAG: --proxyComponentLogLevel="misc:error"
2025-11-23T05:59:12.272656Z     info    FLAG: --proxyLogLevel="warning"
2025-11-23T05:59:12.272658Z     info    FLAG: --serviceCluster="istio-proxy"
2025-11-23T05:59:12.272659Z     info    FLAG: --stsPort="0"
2025-11-23T05:59:12.272661Z     info    FLAG: --templateFile=""
2025-11-23T05:59:12.272662Z     info    FLAG: --tokenManagerPlugin=""
2025-11-23T05:59:12.272664Z     info    FLAG: --vklog="0"
2025-11-23T05:59:12.272666Z     info    Version 1.26.0-c2e9871f340c0e0b114bcd1b73208284f1d17c9e-Clean
2025-11-23T05:59:12.272671Z     info    Set max file descriptors (ulimit -n) to: 1048576
2025-11-23T05:59:12.273004Z     info    Proxy role      ips=[10.50.0.5] type=router id=istio-egressgateway-675cdb9f4b-qngqt.istio-system domain=istio-system.svc.cluster.local
2025-11-23T05:59:12.273084Z     info    Apply mesh config from file accessLogFile: /dev/stdout
defaultConfig:
  discoveryAddress: istiod.istio-system.svc:15012
defaultProviders:
  metrics:
  - prometheus
enablePrometheusMerge: true
extensionProviders:
- envoyOtelAls:
    port: 4317
    service: opentelemetry-collector.observability.svc.cluster.local
  name: otel
- name: skywalking
  skywalking:
    port: 11800
    service: tracing.istio-system.svc.cluster.local
- name: otel-tracing
  opentelemetry:
    port: 4317
    service: opentelemetry-collector.observability.svc.cluster.local
- name: jaeger
  opentelemetry:
    port: 4317
    service: jaeger-collector.istio-system.svc.cluster.local
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
rootNamespace: istio-system
trustDomain: cluster.local
2025-11-23T05:59:12.275003Z     info    cpu limit detected as 2, setting concurrency
2025-11-23T05:59:12.275456Z     info    Effective config: binaryPath: /usr/local/bin/envoy
concurrency: 2
configPath: ./etc/istio/proxy
controlPlaneAuthPolicy: MUTUAL_TLS
discoveryAddress: istiod.istio-system.svc:15012
drainDuration: 45s
proxyAdminPort: 15000
serviceCluster: istio-proxy
statNameLength: 189
statusPort: 15020
terminationDrainDuration: 5s

2025-11-23T05:59:12.275509Z     info    JWT policy is third-party-jwt
2025-11-23T05:59:12.275515Z     info    using credential fetcher of JWT type in cluster.local trust domain
2025-11-23T05:59:12.280223Z     info    platform detected is GCP
2025-11-23T05:59:12.281617Z     info    Starting default Istio SDS Server
2025-11-23T05:59:12.281651Z     info    CA Endpoint istiod.istio-system.svc:15012, provider Citadel
2025-11-23T05:59:12.281670Z     info    Using CA istiod.istio-system.svc:15012 cert with certs: var/run/secrets/istio/root-cert.pem
2025-11-23T05:59:12.281743Z     info    Opening status port 15020
2025-11-23T05:59:12.283683Z     info    xdsproxy        Initializing with upstream address "istiod.istio-system.svc:15012" and cluster "Kubernetes"
2025-11-23T05:59:12.284587Z     info    sds     Starting SDS grpc server
2025-11-23T05:59:12.284627Z     info    sds     Starting SDS server for workload certificates, will listen on "var/run/secrets/workload-spiffe-uds/socket"
2025-11-23T05:59:12.289107Z     warn    Failed to load the zone name of the pod from resolv.conf
2025-11-23T05:59:12.291082Z     info    Pilot SAN: [istiod.istio-system.svc]
2025-11-23T05:59:12.293332Z     info    Starting proxy agent
2025-11-23T05:59:12.293427Z     info    Envoy command: [-c etc/istio/proxy/envoy-rev.json --drain-time-s 45 --drain-strategy immediate --local-address-ip-version v4 --file-flush-interval-msec 1000 --disable-hot-restart --allow-unknown-static-fields -l warning --component-log-level misc:error --skip-deprecated-logs --concurrency 2]
2025-11-23T05:59:12.420156Z     warning envoy main external/envoy/source/server/server.cc:874   Usage of the deprecated runtime key overload.global_downstream_max_connections, consider switching to `envoy.resource_monitors.global_downstream_max_connections` instead.This runtime key will be removed in future.   thread=16
2025-11-23T05:59:12.421195Z     warning envoy main external/envoy/source/server/server.cc:970   There is no configured limit to the number of allowed active downstream connections. Configure a limit in `envoy.resource_monitors.global_downstream_max_connections` resource monitor. thread=16
2025-11-23T05:59:12.429001Z     info    xdsproxy        connected to delta upstream XDS server: istiod.istio-system.svc:15012       id=1
2025-11-23T05:59:12.470781Z     info    ads     ADS: new connection for node:1
2025-11-23T05:59:12.473147Z     info    ads     ADS: new connection for node:2
2025-11-23T05:59:12.531392Z     info    cache   generated new workload certificate      resourceName=default latency=248.203666ms ttl=23h59m59.468612655s
2025-11-23T05:59:12.531750Z     info    cache   Root cert has changed, start rotating root cert
2025-11-23T05:59:12.531801Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.468199691s
2025-11-23T05:59:12.531852Z     info    cache   returned workload certificate from cache        ttl=23h59m59.468148675s
2025-11-23T05:59:12.532062Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.467943051s
2025-11-23T05:59:12.532463Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.467538251s
2025-11-23T05:59:13.055578Z     info    Readiness succeeded in 791.008861ms
2025-11-23T05:59:13.062235Z     info    Envoy proxy is ready


While the previous command is running, execute the following commands to send requests to Google and Wikipedia:

kubectl exec -ti test -- curl --head -L http://www.google.com
HTTP/1.1 502 Bad Gateway
date: Sun, 23 Nov 2025 06:13:21 GMT
server: envoy
transfer-encoding: chunked

kubectl exec -ti test -- curl --head -L http://www.wikipedia.org
HTTP/1.1 301 Moved Permanently
content-length: 0
location: https://www.wikipedia.org/
server: envoy
x-cache: cp2039 int
x-cache-status: int-tls
x-envoy-upstream-service-time: 52
date: Sun, 23 Nov 2025 06:13:40 GMT

HTTP/2 200 
date: Sat, 22 Nov 2025 18:17:43 GMT
cache-control: s-maxage=86400, must-revalidate, max-age=3600
server: ATS/9.2.11
etag: W/"1bb6c-6426794fdf540"
last-modified: Thu, 30 Oct 2025 22:15:09 GMT
content-type: text/html
age: 42957
accept-ranges: bytes
x-cache: cp2041 hit, cp2031 hit/440833
x-cache-status: hit-front
server-timing: cache;desc="hit-front", host;desc="cp2031"
strict-transport-security: max-age=106384710; includeSubDomains; preload
report-to: { "group": "wm_nel", "max_age": 604800, "endpoints": [{ "url": "https://intake-logging.wikimedia.org/v1/events?stream=w3c.reportingapi.network_error&schema_uri=/w3c/reportingapi/network_error/1.0.0" }] }
nel: { "report_to": "wm_nel", "max_age": 604800, "failure_fraction": 0.05, "success_fraction": 0.0}
set-cookie: WMF-Last-Access=23-Nov-2025;Path=/;HttpOnly;secure;Expires=Thu, 25 Dec 2025 00:00:00 GMT
set-cookie: WMF-Last-Access-Global=23-Nov-2025;Path=/;Domain=.wikipedia.org;HttpOnly;secure;Expires=Thu, 25 Dec 2025 00:00:00 GMT
x-client-ip: 35.188.139.128
set-cookie: GeoIP=US:IA:Council_Bluffs:41.26:-95.85:v4; Path=/; secure; Domain=.wikipedia.org
set-cookie: NetworkProbeLimit=0.001;Path=/;Secure;SameSite=None;Max-Age=3600
set-cookie: WMF-Uniq=tyKE4ROTYjA0I-jPPhfIDwK0AAAAAFvdwenlEUpnLgI-_y_VA9YXexif-20aFqwj;Domain=.wikipedia.org;Path=/;HttpOnly;secure;SameSite=None;Expires=Mon, 23 Nov 2026 00:00:00 GMT
content-length: 113516
x-request-id: 8a6ac9d3-35fa-45ac-aeb5-16b2af7c859c
x-analytics: 

As a result, you should not see any logging in the Egress Gateway logs; this indicates that traffic is not currently being routed through the Egress Gateway.

To ensure all outgoing traffic flows through the Egress Gateway, you will need to implement an Egress Gateway alongside a Service Entry. This requires the creation of three resources: a Gateway, a Virtual Service, and a Destination Rule.

Configure external HTTP traffic to www.wikipedia.org to flow through the Istio Egress Gateway instead of going directly to the internet.
You must create and apply the following Istio resources using the specified names:

1) A Gateway named istio-egressgateway

The Gateway must listen on port 80 for HTTP traffic targeting www.wikipedia.org.
2) A DestinationRule named egressgateway-for-wikipedia

Use subset wikipedia in the Destination Rule to route traffic through the Gateway.
3) A Virtual Service named wikipedia-egress-gateway

The Virtual Service must:
Handle traffic from both the mesh and the istio-egressgateway gateways.
Route incoming HTTP traffic from within the mesh to the egress gateway.
Forward traffic from the gateway to www.wikipedia.org.
Make sure to define all resources so that traffic to www.wikipedia.org from within the mesh is routed through the Egress Gateway.

kubectl apply -f egressgateway.yaml 
gateway.networking.istio.io/istio-egressgateway created

kubectl apply -f virtualService.yaml 
virtualservice.networking.istio.io/wikipedia-egress-gateway created

kubectl apply -f destinationRules.yaml 
destinationrule.networking.istio.io/egressgateway-for-wikipedia created

kubectl get vs -A
NAMESPACE   NAME                       GATEWAYS                         HOSTS                   AGE
default     wikipedia-egress-gateway   ["istio-egressgateway","mesh"]   ["www.wikipedia.org"]   31s

Please verify once again that the Egress Gateway is correctly handling traffic by reviewing its logs while making requests to Wikipedia.org.

1) To obtain the Gateway Pod name, execute the following command:

kubectl get pod -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-675cdb9f4b-qngqt    1/1     Running   0          22m
istio-ingressgateway-6cd9bc7f5b-ngnl5   1/1     Running   0          22m
istiod-7f898458c5-5c9ld                 1/1     Running   0          22m

 In a separate terminal, run the following command to view the logs:
 kubectl logs -f -n istio-system istio-egressgateway-675cdb9f4b-qngqt
2025-11-23T05:59:12.272578Z     info    FLAG: --concurrency="0"
2025-11-23T05:59:12.272630Z     info    FLAG: --domain="istio-system.svc.cluster.local"
2025-11-23T05:59:12.272635Z     info    FLAG: --help="false"
2025-11-23T05:59:12.272637Z     info    FLAG: --log_as_json="false"
2025-11-23T05:59:12.272638Z     info    FLAG: --log_caller=""
2025-11-23T05:59:12.272640Z     info    FLAG: --log_output_level="default:info"
2025-11-23T05:59:12.272642Z     info    FLAG: --log_stacktrace_level="default:none"
2025-11-23T05:59:12.272648Z     info    FLAG: --log_target="[stdout]"
2025-11-23T05:59:12.272650Z     info    FLAG: --meshConfig="./etc/istio/config/mesh"
2025-11-23T05:59:12.272651Z     info    FLAG: --outlierLogPath=""
2025-11-23T05:59:12.272653Z     info    FLAG: --profiling="true"
2025-11-23T05:59:12.272654Z     info    FLAG: --proxyComponentLogLevel="misc:error"
2025-11-23T05:59:12.272656Z     info    FLAG: --proxyLogLevel="warning"
2025-11-23T05:59:12.272658Z     info    FLAG: --serviceCluster="istio-proxy"
2025-11-23T05:59:12.272659Z     info    FLAG: --stsPort="0"
2025-11-23T05:59:12.272661Z     info    FLAG: --templateFile=""
2025-11-23T05:59:12.272662Z     info    FLAG: --tokenManagerPlugin=""
2025-11-23T05:59:12.272664Z     info    FLAG: --vklog="0"
2025-11-23T05:59:12.272666Z     info    Version 1.26.0-c2e9871f340c0e0b114bcd1b73208284f1d17c9e-Clean
2025-11-23T05:59:12.272671Z     info    Set max file descriptors (ulimit -n) to: 1048576
2025-11-23T05:59:12.273004Z     info    Proxy role      ips=[10.50.0.5] type=router id=istio-egressgateway-675cdb9f4b-qngqt.istio-system domain=istio-system.svc.cluster.local
2025-11-23T05:59:12.273084Z     info    Apply mesh config from file accessLogFile: /dev/stdout
defaultConfig:
  discoveryAddress: istiod.istio-system.svc:15012
defaultProviders:
  metrics:
  - prometheus
enablePrometheusMerge: true
extensionProviders:
- envoyOtelAls:
    port: 4317
    service: opentelemetry-collector.observability.svc.cluster.local
  name: otel
- name: skywalking
  skywalking:
    port: 11800
    service: tracing.istio-system.svc.cluster.local
- name: otel-tracing
  opentelemetry:
    port: 4317
    service: opentelemetry-collector.observability.svc.cluster.local
- name: jaeger
  opentelemetry:
    port: 4317
    service: jaeger-collector.istio-system.svc.cluster.local
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
rootNamespace: istio-system
trustDomain: cluster.local
2025-11-23T05:59:12.275003Z     info    cpu limit detected as 2, setting concurrency
2025-11-23T05:59:12.275456Z     info    Effective config: binaryPath: /usr/local/bin/envoy
concurrency: 2
configPath: ./etc/istio/proxy
controlPlaneAuthPolicy: MUTUAL_TLS
discoveryAddress: istiod.istio-system.svc:15012
drainDuration: 45s
proxyAdminPort: 15000
serviceCluster: istio-proxy
statNameLength: 189
statusPort: 15020
terminationDrainDuration: 5s

2025-11-23T05:59:12.275509Z     info    JWT policy is third-party-jwt
2025-11-23T05:59:12.275515Z     info    using credential fetcher of JWT type in cluster.local trust domain
2025-11-23T05:59:12.280223Z     info    platform detected is GCP
2025-11-23T05:59:12.281617Z     info    Starting default Istio SDS Server
2025-11-23T05:59:12.281651Z     info    CA Endpoint istiod.istio-system.svc:15012, provider Citadel
2025-11-23T05:59:12.281670Z     info    Using CA istiod.istio-system.svc:15012 cert with certs: var/run/secrets/istio/root-cert.pem
2025-11-23T05:59:12.281743Z     info    Opening status port 15020
2025-11-23T05:59:12.283683Z     info    xdsproxy        Initializing with upstream address "istiod.istio-system.svc:15012" and cluster "Kubernetes"
2025-11-23T05:59:12.284587Z     info    sds     Starting SDS grpc server
2025-11-23T05:59:12.284627Z     info    sds     Starting SDS server for workload certificates, will listen on "var/run/secrets/workload-spiffe-uds/socket"
2025-11-23T05:59:12.289107Z     warn    Failed to load the zone name of the pod from resolv.conf
2025-11-23T05:59:12.291082Z     info    Pilot SAN: [istiod.istio-system.svc]
2025-11-23T05:59:12.293332Z     info    Starting proxy agent
2025-11-23T05:59:12.293427Z     info    Envoy command: [-c etc/istio/proxy/envoy-rev.json --drain-time-s 45 --drain-strategy immediate --local-address-ip-version v4 --file-flush-interval-msec 1000 --disable-hot-restart --allow-unknown-static-fields -l warning --component-log-level misc:error --skip-deprecated-logs --concurrency 2]
2025-11-23T05:59:12.420156Z     warning envoy main external/envoy/source/server/server.cc:874   Usage of the deprecated runtime key overload.global_downstream_max_connections, consider switching to `envoy.resource_monitors.global_downstream_max_connections` instead.This runtime key will be removed in future.   thread=16
2025-11-23T05:59:12.421195Z     warning envoy main external/envoy/source/server/server.cc:970   There is no configured limit to the number of allowed active downstream connections. Configure a limit in `envoy.resource_monitors.global_downstream_max_connections` resource monitor. thread=16
2025-11-23T05:59:12.429001Z     info    xdsproxy        connected to delta upstream XDS server: istiod.istio-system.svc:15012       id=1
2025-11-23T05:59:12.470781Z     info    ads     ADS: new connection for node:1
2025-11-23T05:59:12.473147Z     info    ads     ADS: new connection for node:2
2025-11-23T05:59:12.531392Z     info    cache   generated new workload certificate      resourceName=default latency=248.203666ms ttl=23h59m59.468612655s
2025-11-23T05:59:12.531750Z     info    cache   Root cert has changed, start rotating root cert
2025-11-23T05:59:12.531801Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.468199691s
2025-11-23T05:59:12.531852Z     info    cache   returned workload certificate from cache        ttl=23h59m59.468148675s
2025-11-23T05:59:12.532062Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.467943051s
2025-11-23T05:59:12.532463Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.467538251s
2025-11-23T05:59:13.055578Z     info    Readiness succeeded in 791.008861ms
2025-11-23T05:59:13.062235Z     info    Envoy proxy is ready

To tail the pod logs and make a request to Wikipedia, use the following command:
 
kubectl exec -ti test -- curl --head -L http://www.wikipedia.org
HTTP/1.1 301 Moved Permanently
content-length: 0
location: https://www.wikipedia.org/
server: envoy
x-cache: cp2039 int
x-cache-status: int-tls
x-envoy-upstream-service-time: 53
date: Sun, 23 Nov 2025 06:24:53 GMT

HTTP/2 200 
date: Sat, 22 Nov 2025 18:17:43 GMT
cache-control: s-maxage=86400, must-revalidate, max-age=3600
server: ATS/9.2.11
etag: W/"1bb6c-6426794fdf540"
last-modified: Thu, 30 Oct 2025 22:15:09 GMT
content-type: text/html
age: 43630
accept-ranges: bytes
x-cache: cp2041 hit, cp2031 hit/447220
x-cache-status: hit-front
server-timing: cache;desc="hit-front", host;desc="cp2031"
strict-transport-security: max-age=106384710; includeSubDomains; preload
report-to: { "group": "wm_nel", "max_age": 604800, "endpoints": [{ "url": "https://intake-logging.wikimedia.org/v1/events?stream=w3c.reportingapi.network_error&schema_uri=/w3c/reportingapi/network_error/1.0.0" }] }
nel: { "report_to": "wm_nel", "max_age": 604800, "failure_fraction": 0.05, "success_fraction": 0.0}
set-cookie: WMF-Last-Access=23-Nov-2025;Path=/;HttpOnly;secure;Expires=Thu, 25 Dec 2025 00:00:00 GMT
set-cookie: WMF-Last-Access-Global=23-Nov-2025;Path=/;Domain=.wikipedia.org;HttpOnly;secure;Expires=Thu, 25 Dec 2025 00:00:00 GMT
x-client-ip: 35.188.139.128
set-cookie: GeoIP=US:IA:Council_Bluffs:41.26:-95.85:v4; Path=/; secure; Domain=.wikipedia.org
set-cookie: NetworkProbeLimit=0.001;Path=/;Secure;SameSite=None;Max-Age=3600
set-cookie: WMF-Uniq=NfEi58cXp-_Y0-UFym7Z-AK0AAAAAFvd_s92ZMcai1yoTLWxajJs_-2xeIaTy5Qs;Domain=.wikipedia.org;Path=/;HttpOnly;secure;SameSite=None;Expires=Mon, 23 Nov 2026 00:00:00 GMT
content-length: 113516
x-request-id: a35e45f7-7333-43c6-b2cc-5f182b57b376
x-analytics: 


kubectl logs -f -n istio-system istio-egressgateway-675cdb9f4b-qngqt
2025-11-23T05:59:12.272578Z     info    FLAG: --concurrency="0"
2025-11-23T05:59:12.272630Z     info    FLAG: --domain="istio-system.svc.cluster.local"
2025-11-23T05:59:12.272635Z     info    FLAG: --help="false"
2025-11-23T05:59:12.272637Z     info    FLAG: --log_as_json="false"
2025-11-23T05:59:12.272638Z     info    FLAG: --log_caller=""
2025-11-23T05:59:12.272640Z     info    FLAG: --log_output_level="default:info"
2025-11-23T05:59:12.272642Z     info    FLAG: --log_stacktrace_level="default:none"
2025-11-23T05:59:12.272648Z     info    FLAG: --log_target="[stdout]"
2025-11-23T05:59:12.272650Z     info    FLAG: --meshConfig="./etc/istio/config/mesh"
2025-11-23T05:59:12.272651Z     info    FLAG: --outlierLogPath=""
2025-11-23T05:59:12.272653Z     info    FLAG: --profiling="true"
2025-11-23T05:59:12.272654Z     info    FLAG: --proxyComponentLogLevel="misc:error"
2025-11-23T05:59:12.272656Z     info    FLAG: --proxyLogLevel="warning"
2025-11-23T05:59:12.272658Z     info    FLAG: --serviceCluster="istio-proxy"
2025-11-23T05:59:12.272659Z     info    FLAG: --stsPort="0"
2025-11-23T05:59:12.272661Z     info    FLAG: --templateFile=""
2025-11-23T05:59:12.272662Z     info    FLAG: --tokenManagerPlugin=""
2025-11-23T05:59:12.272664Z     info    FLAG: --vklog="0"
2025-11-23T05:59:12.272666Z     info    Version 1.26.0-c2e9871f340c0e0b114bcd1b73208284f1d17c9e-Clean
2025-11-23T05:59:12.272671Z     info    Set max file descriptors (ulimit -n) to: 1048576
2025-11-23T05:59:12.273004Z     info    Proxy role      ips=[10.50.0.5] type=router id=istio-egressgateway-675cdb9f4b-qngqt.istio-system domain=istio-system.svc.cluster.local
2025-11-23T05:59:12.273084Z     info    Apply mesh config from file accessLogFile: /dev/stdout
defaultConfig:
  discoveryAddress: istiod.istio-system.svc:15012
defaultProviders:
  metrics:
  - prometheus
enablePrometheusMerge: true
extensionProviders:
- envoyOtelAls:
    port: 4317
    service: opentelemetry-collector.observability.svc.cluster.local
  name: otel
- name: skywalking
  skywalking:
    port: 11800
    service: tracing.istio-system.svc.cluster.local
- name: otel-tracing
  opentelemetry:
    port: 4317
    service: opentelemetry-collector.observability.svc.cluster.local
- name: jaeger
  opentelemetry:
    port: 4317
    service: jaeger-collector.istio-system.svc.cluster.local
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
rootNamespace: istio-system
trustDomain: cluster.local
2025-11-23T05:59:12.275003Z     info    cpu limit detected as 2, setting concurrency
2025-11-23T05:59:12.275456Z     info    Effective config: binaryPath: /usr/local/bin/envoy
concurrency: 2
configPath: ./etc/istio/proxy
controlPlaneAuthPolicy: MUTUAL_TLS
discoveryAddress: istiod.istio-system.svc:15012
drainDuration: 45s
proxyAdminPort: 15000
serviceCluster: istio-proxy
statNameLength: 189
statusPort: 15020
terminationDrainDuration: 5s

2025-11-23T05:59:12.275509Z     info    JWT policy is third-party-jwt
2025-11-23T05:59:12.275515Z     info    using credential fetcher of JWT type in cluster.local trust domain
2025-11-23T05:59:12.280223Z     info    platform detected is GCP
2025-11-23T05:59:12.281617Z     info    Starting default Istio SDS Server
2025-11-23T05:59:12.281651Z     info    CA Endpoint istiod.istio-system.svc:15012, provider Citadel
2025-11-23T05:59:12.281670Z     info    Using CA istiod.istio-system.svc:15012 cert with certs: var/run/secrets/istio/root-cert.pem
2025-11-23T05:59:12.281743Z     info    Opening status port 15020
2025-11-23T05:59:12.283683Z     info    xdsproxy        Initializing with upstream address "istiod.istio-system.svc:15012" and cluster "Kubernetes"
2025-11-23T05:59:12.284587Z     info    sds     Starting SDS grpc server
2025-11-23T05:59:12.284627Z     info    sds     Starting SDS server for workload certificates, will listen on "var/run/secrets/workload-spiffe-uds/socket"
2025-11-23T05:59:12.289107Z     warn    Failed to load the zone name of the pod from resolv.conf
2025-11-23T05:59:12.291082Z     info    Pilot SAN: [istiod.istio-system.svc]
2025-11-23T05:59:12.293332Z     info    Starting proxy agent
2025-11-23T05:59:12.293427Z     info    Envoy command: [-c etc/istio/proxy/envoy-rev.json --drain-time-s 45 --drain-strategy immediate --local-address-ip-version v4 --file-flush-interval-msec 1000 --disable-hot-restart --allow-unknown-static-fields -l warning --component-log-level misc:error --skip-deprecated-logs --concurrency 2]
2025-11-23T05:59:12.420156Z     warning envoy main external/envoy/source/server/server.cc:874   Usage of the deprecated runtime key overload.global_downstream_max_connections, consider switching to `envoy.resource_monitors.global_downstream_max_connections` instead.This runtime key will be removed in future.   thread=16
2025-11-23T05:59:12.421195Z     warning envoy main external/envoy/source/server/server.cc:970   There is no configured limit to the number of allowed active downstream connections. Configure a limit in `envoy.resource_monitors.global_downstream_max_connections` resource monitor. thread=16
2025-11-23T05:59:12.429001Z     info    xdsproxy        connected to delta upstream XDS server: istiod.istio-system.svc:15012       id=1
2025-11-23T05:59:12.470781Z     info    ads     ADS: new connection for node:1
2025-11-23T05:59:12.473147Z     info    ads     ADS: new connection for node:2
2025-11-23T05:59:12.531392Z     info    cache   generated new workload certificate      resourceName=default latency=248.203666ms ttl=23h59m59.468612655s
2025-11-23T05:59:12.531750Z     info    cache   Root cert has changed, start rotating root cert
2025-11-23T05:59:12.531801Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.468199691s
2025-11-23T05:59:12.531852Z     info    cache   returned workload certificate from cache        ttl=23h59m59.468148675s
2025-11-23T05:59:12.532062Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.467943051s
2025-11-23T05:59:12.532463Z     info    cache   returned workload trust anchor from cache       ttl=23h59m59.467538251s
2025-11-23T05:59:13.055578Z     info    Readiness succeeded in 791.008861ms
2025-11-23T05:59:13.062235Z     info    Envoy proxy is ready
[2025-11-23T06:23:14.346Z] "GET / HTTP/2" 301 - via_upstream - "-" 0 0 52 52 "10.50.0.7" "curl/8.14.1" "06024f7f-3000-975c-b514-7afbb111fcee" "www.wikipedia.org" "208.80.153.224:80" outbound|80||www.wikipedia.org 10.50.0.5:34748 10.50.0.5:8080 10.50.0.7:37434 - -
[2025-11-23T06:23:34.864Z] "GET / HTTP/2" 301 - via_upstream - "-" 0 0 51 51 "10.50.0.7" "curl/8.14.1" "0abd2783-0d03-9e01-896c-9c1b743b0f63" "www.wikipedia.org" "208.80.153.224:80" outbound|80||www.wikipedia.org 10.50.0.5:58338 10.50.0.5:8080 10.50.0.7:37434 - -


