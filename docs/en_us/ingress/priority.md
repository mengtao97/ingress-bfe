# Priority of route rules
If a request matches multiple ingress rules, BFE Ingress Controller will decide which rule will be hit according to below strategies:

-  Compare the hostname and select the rule with most precise hostname;
-  If more than one rule is selected in the above step, select the rule with most precise path;
-  If more than one rule is selected in the above step, select the rule with most advanced conditions;
-  If more than one rule is selected in the above step, select the rule which matches an advanced condition of higher priority
   - in advanced condition, Cookie condition has higher priority than Header condition；

## Examples
### Hostname precision first
```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1beta1
metadata:
  name: "host_priority1"
  namespace: production
  annotations:
    kubernetes.io/ingress.class: bfe 
spec:
  rules:
    - host: example.net
      http:
        paths:
          - path: /bar
            backend:
              serviceName: service1
              servicePort: 80
---
kind: Ingress
apiVersion: networking.k8s.io/v1beta1
metadata:
  name: "host_priority2"
  namespace: production
  annotations:
    kubernetes.io/ingress.class: bfe 
spec:
  rules:
    - host: *.net
      http:
        paths:
          - path: /bar
            backend:
              serviceName: service2
              servicePort: 80
```
In above example, for requests generated by `curl "http://example.net/bar"`, the rule in ingress `host_priority1` will be hit.

### Path precision first when hostname are identical
```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1beta1
metadata:
  name: "path_priority1"
  namespace: production
  annotations:
    kubernetes.io/ingress.class: bfe 
spec:
  rules:
    - host: example.net
      http:
        paths:
          - path: /bar/foo
            backend:
              serviceName: service1
              servicePort: 80
---
kind: Ingress
apiVersion: networking.k8s.io/v1beta1
metadata:
  name: "path_priority2"
  namespace: production
  annotations:
    kubernetes.io/ingress.class: bfe 
    bfe.ingress.kubernetes.io/router.header: "key: value"
spec:
  rules:
    - host: example.net
      http:
        paths:
          - path: /bar
            backend:
              serviceName: service2
              servicePort: 80
```
In above example, for requests generated by `curl "http://example.net/bar/foo" -H "Key: value"`, the rule in ingress `path_priority1` will be hit

### More advanced condition first, when hostname and path both identical
```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1beta1
metadata:
  name: "cond_priority1"
  namespace: production
  annotations:
    kubernetes.io/ingress.class: bfe 
    bfe.ingress.kubernetes.io/router.header: "key: value"
spec:
  rules:
    - host: example.net
      http:
        paths:
          - path: /bar
            backend:
              serviceName: service1
              servicePort: 80
---
kind: Ingress
apiVersion: networking.k8s.io/v1beta1
metadata:
  name: "cond_priority1"
  namespace: production  
  annotations:
    kubernetes.io/ingress.class: bfe 
  
spec:
  rules:
    - host: example.net
      http:
        paths:
          - path: /bar
            backend:
              serviceName: service2
              servicePort: 80
```
In above example, for requests generated by `curl "http://example.net/bar/foo" -H "Key: value"`, the rule in ingress `cond_priority1` will be hit

### Matched advanced condition with higher priority first, when multiple rules meet above criteria
```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1beta1
metadata:
  name: "multi_cond_priority1"
  namespace: production
  annotations:
    kubernetes.io/ingress.class: bfe 
    bfe.ingress.kubernetes.io/router.header: "header-key: value"
spec:
  rules:
    - host: example.net
      http:
        paths:
          - path: /bar
            backend:
              serviceName: service1
              servicePort: 80
---
kind: Ingress
apiVersion: networking.k8s.io/v1beta1
metadata:
  name: "multi_cond_priority2"
  namespace: production
  annotations:
    kubernetes.io/ingress.class: bfe 
    bfe.ingress.kubernetes.io/router.cookie: "cookie-key: value"
spec:
  rules:
    - host: example.net
      http:
        paths:
          - path: /bar
            backend:
              serviceName: service2
              servicePort: 80
```
In above example, for requests generated by `curl "http://example.net/bar/foo" -H "Header-key: value" --cookie "cookie-key: value"`, the rule in ingress `multi_cond_priority2` will be hit, because `Cookie` condition has higher priority than `Header` condition.
