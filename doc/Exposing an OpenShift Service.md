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
Expose services running in OpenShift outside the scope of the cluster.

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

