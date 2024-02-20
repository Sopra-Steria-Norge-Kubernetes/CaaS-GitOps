---
Author: Eirik Opheim
Title: Exposing an openshift service
Version: 0.0.1
Target Audience: Developers
Dependencies: None
externally-exposed: true
---
[[_TOC_]]

# Exposing an OpenShift service

## Purpose
Expose services running in OpenShift outside the scope of the cluster. For documentation on how to create OpenShift routes, please refer to the [OpenShift documentation](https://docs.openshift.com/container-platform/4.13/networking/routes/route-configuration.html)
### Exposing a service using the wildcard ingress certificate

To expose a service running in OpenShift you can create an object of type Ingress. 

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress (1)
  namespace: my-namespace (2)
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    haproxy.router.openshift.io/ip_whitelist: '0.0.0.0/0' (3)
spec:
  tls: (4)
  - {}
  ingressClassName: openshift-default (5)
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test (6)
            port:
              number: 80 (7)
```

1. Name of your ingress object.
2. Namespace where your ingress object is built.
3. IP adress range allowed to reach your service. `0.0.0.0/0`will allow all source IP's.
4. TLS configuration. The configuration in the example will use the wildcard ingress certificate to expose your service. 
5. Which ingress class should be used to expose your service.
6. Name of your service.
7. Port exposed by your service.

### Expose services on a different ingressController

While exposing services on a specific ingressController you have to set two additional fields as seen below. 

```
kind: Ingress
apiVersion: networking.k8s.io/v1 
metadata: 
  name: minimal-ingress
  namespace: my-namespace 
  labels: 
    mydeveloperplatform/exposetointernet: "true" (1) 
spec:
  ingressClassName: openshift-ingress-external (2)
```

1. The label used to define that your route should be exposed on a specific ingress. Which label should be used for each customer is agreed upon once self-service of exposing services on a specific ingress is approved.
2. The name of the ingress class you are using. Also agreed upon when exposing services on a specific ingress is set up.

### Requirements for all ingress objects

All ingress objects are required to specify:
```
metadata:
  annotations:
    haproxy.router.openshift.io/ip_whitelist: 
```
Reason: Developers can expose their services wherever they like, but they need to actively decide which IP range should be allowed to access their service.

```
spec:
  tls:
```
Reason: Services should be exposed using HTTPS, not H

