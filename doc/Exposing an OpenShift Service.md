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
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    haproxy.router.openshift.io/ip_whitelist: '0.0.0.0/0'
spec:
  tls: 
  - hosts:
    - myhost 
  ingressClassName: openshift-default
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
