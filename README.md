# sockshop-istio
Sockshop demo with Istio service mesh

## File details
|Files|Description|
|------|--------|
|Sock-shop-complete-demo.yaml| contains all the changes required in sock-shop deployment to be compatible with Istio.|
|sock-shop-complete-demo-istio-1.yaml| sock-shop deployment yaml spec with manually injected Istio|                                                      
                                                                                                                                                      
## Installation/Setup                                                                                                                                 
                                                                                                                                                      
1. Install Istio-1.0.4 using helm charts.                                                                                                             
2. Run bellow command due for `catalogue` service to be able to connect to `catalogue-db`. More info : https://github.com/istio/istio/issues/10062    
                                                                                                                                                      
```                                                                                                                                                   
  kubectl delete meshpolicies.authentication.istio.io default                                                                                         
```                                                                                                                                                   
3. Deploy sock-shop application.                                                                                                                      
                                                                                                                                                      
```                                                                                                                                                   
  kubectl apply -f sock-shop-complete-demo-istio-1.yaml
```  
