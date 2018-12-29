# sockshop-istio
Sockshop demo with Istio service mesh

- [sockshop-istio](#sockshop-istio)
  * [File details](#file-details)
  * [1. Installation/Setup](#1-installation-setup)
  * [2. Intelligent Routing (Canary Deployment)](#2-intelligent-routing--canary-deployment-)
  * [3. Circuit Breaker Pattern](#3-circuit-breaker-pattern)
  * [4. Security - Mutual TLS](#4-security---mutual-tls)
  * [5. Timeouts/Fault Injection](#5-timeouts-fault-injection)
  * [6. Telemetry](#6-telemetry)
  
## File details
|Files|Description|
|------|--------|
|[sock-shop-complete-demo.yaml](https://github.com/infracloudio/sockshop-istio/blob/master/sock-shop-complete-demo.yaml)| contains all the changes required in sock-shop deployment to be compatible with Istio.|
|[sock-shop-complete-demo-istio-1.yaml](https://github.com/infracloudio/sockshop-istio/blob/master/sock-shop-complete-demo-istio-1.yaml)| sock-shop deployment yaml spec with manually injected Istio|

## 1. Installation/Setup                                                                                                                                 

1. Install Istio-1.0.4 using helm charts.
2. Run bellow command due for `catalogue` service to be able to connect to `catalogue-db`. More info : https://github.com/istio/istio/issues/10062

```                                                                                                                                                   
  kubectl delete meshpolicies.authentication.istio.io default
```
3. Deploy sock-shop application.

```                                                                                                                                                   
  kubectl apply -f sock-shop-complete-demo-istio-1.yaml
```
## 2. Intelligent Routing (Canary Deployment)
1. Apply version 2 of fron-end.
```
kubectl apply -f 2-front-end-deployment-v2-istio.yaml -nsock-shop
```
2. Update `front-end` istio VirtualService to route traffic to `front-end-v2`.
```
istioctl replace -f 2-front-end-deployment-v2-route.yaml -nsock-shop
```

## 3. Circuit Breaker Pattern

1. Run Fortio app with 3 connections and 20 requests. See all requests go through
```
kubectl apply -f 4-fortio-deploy-istio.yaml
FORTIO_POD=$(kubectl get pod -nsock-shop | grep fortio | awk '{ print $1 }')
kubectl exec -it $FORTIO_POD  -nsock-shop -c fortio /usr/local/bin/fortio -- load -curl  http://front-end:80/index.html
kubectl exec -it $FORTIO_POD -nsock-shop -c fortio /usr/local/bin/fortio -- load -c 3 -qps 0 -n 20 -loglevel Warning http://front-end:80/index.html
```
2. Apply circuit breaker destination rule for max 1 connection.
```
kubectl apply -f 4-circuit-breaker.yaml
```
2. Run Fortio app with 3 connections and 20 requests. 30% should pass and 70% should fail.
```
kubectl exec -it $FORTIO_POD -nsock-shop -c fortio /usr/local/bin/fortio -- load -c 3 -qps 0 -n 20 -loglevel Warning http://front-end:80/index.html
```
3. Update destination rule for max 2 concurrent connections.
4. Run Fortio app with 3 connections and 20 requests. 70% should pass and 30% should fail.
```
kubectl exec -it $FORTIO_POD -nsock-shop -c fortio /usr/local/bin/fortio -- load -c 3 -qps 0 -n 20 -loglevel Warning http://front-end:80/index.html
```
  
## 4. Security - Mutual TLS

1. Apply mesh-wide authentication policy in `default` namespace. This will enable all the receiving (server) sides of the service to use TLS.
```
istioctl create -f 5-global-mtls-mesh-policy.yaml
```
2. Load the front-end in browser. See that catalogues are not loading since `catalogue` service is rejecting plain-text `front-end` connections.
3. Update all the destination-rules to use TLS. This will enable all the sender (client) sides of the services to use TLS.
```
istioctl replace -f 5-virtual-services-all-mtls.yaml
```
4. Load the front-end again. See that its fuctioning properly now.
5. Verify that certs are automatically injected into sidecar proxies
```
kubectl exec -nsock-shop -c istio-proxy carts-66469c84c6-jj2zt -- ls /etc/certs
cert-chain.pem    <-- cert to be presented to other side   
key.pem           <-- side cars private key
root-cert.pem     <-- root cert to verify peer's cert
```
6. Verify using istioctl
```
istioctl authn tls-check carts.sock-shop.svc.cluster.local -nsock-shop
HOST:PORT                                STATUS     SERVER     CLIENT     AUTHN POLICY     DESTINATION RULE
carts.sock-shop.svc.cluster.local:80     OK         mTLS       mTLS       default/         carts/sock-shop
                                                                          ^Default mesh    ^Destination rule
                                                                          policy since
                                                                          namespace is blanck
```
## 5. Timeouts/Fault Injection

## 6. Telemetry

### Prometheus
1. Connect to prometheus
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &
```
2. Access Prometheus dashboard
```
http://localhost:9090/graph
```
3. Query for total requests to `catalogue` service
```
istio_requests_total{destination_service="catalogue.sock-shop.svc.cluster.local"}
rate(istio_requests_total{destination_service=~"catalogue.*", response_code="200"}[5m])     <-- HTTP Success Rate to catalgue serivce for last 5 mins
```
### Grafana
1. Connect to Grafana
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```
2. Access Grafana dashboard
```
http://localhost:3000/dashboard/db/istio-mesh-dashboard 
```

### Jaeger
1. Connect to Jaeger
```
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686 &
```

2. Access the dashboard
```
http://localhost:16686
```

### Kiali
1. Connect to Kiali.
```
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001 &
```
2. Access the dashboard (Default username/password: admin/admin)
```
http://localhost:20001/
```
