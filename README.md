# rosa-gateway

This guide shows how to use the Gateway API on a ROSA cluster to
provide access to a 3rd party web service. For full documentation, see
[Ingress and Load
Balancing](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/ingress_and_load_balancing/configuring-ingress-cluster-traffic#ingress-gateway-api).

## Pre-requisites

This guide assumes you have already provisioned a ROSA cluster and
have a user with cluster-admin access. You'll also need a web service
to proxy traffic to.


## Overview

The OpenShift Ingress Operator already includes support for Gateway
API, using an [Istio](https://istio.io/) controller and the [Envoy
Proxy](https://www.envoyproxy.io/).

! See the important note about OpenShift Service Mesh.

In this example, we'll be proxying traffic to redhat.com from an
endpoint on our ROSA cluster. In a real world scenario, you'd be using
this proxy to provide centralized access to an internal service.

## Implementation

### GatewayClass

First, create the GatewayClass object. This tells the Ingress Operator
to provision the required infrastructure for hosting a Gateway.

```
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: openshift-default
spec:
  controllerName: openshift.io/gateway-controller/v1
```

It is important not to modify the name of the GatewayClass. The
Ingress Operator will only manage a GatewayClass named "openshift-default".

Then, verify that the GatewayClass was accepted by the Ingress Operator:

```
$ oc get gatewayclass -o yaml
apiVersion: v1
items:
- apiVersion: gateway.networking.k8s.io/v1
  kind: GatewayClass
  metadata:
    creationTimestamp: "2026-05-29T14:00:32Z"
    generation: 1
    name: openshift-default
    resourceVersion: "264150"
    uid: 8f87fc80-1647-4646-8125-390dd43e1bc0
  spec:
    controllerName: openshift.io/gateway-controller/v1
  status:
    conditions:
    - lastTransitionTime: "2026-05-29T14:01:06Z"
      message: Handled by Istio controller
      observedGeneration: 1
      reason: Accepted
      status: "True"
      type: Accepted
```

You should now see an Istio pod running in the ```openshift-ingress``` namespace:

```
$ oc get pods -n openshift-ingress
NAME                                        READY   STATUS    RESTARTS   AGE
istiod-openshift-gateway-767cf5cf55-8g74x   1/1     Running   0          3m4s
router-default-5778c4f6b5-9xgs6             1/1     Running   0          24h
router-default-5778c4f6b5-v6slx             1/1     Running   0          24h
```

Next, download your ingress wildcard certificates and save them as a secret in the ```openshift-ingress``` namespace:

```
$ oc get secret/default-ingress-cert -n openshift-ingress -o jsonpath='{.data.tls\.key}'| base64 -d > /tmp/default.key
$ oc get secret/default-ingress-cert -n openshift-ingress -o jsonpath='{.data.tls\.crt}'| base64 -d > /tmp/default.cert
# echo the certifiate and key to base64 -d into two files
# then create the secret:
$ oc -n openshift-ingress create secret tls gateway-wildcard --cert=/tmp/default.cert --key=/tmp/default.key 
secret/gateway-wildcard created
```

### Gateway

Create a wildcard Gateway object. Make sure to replace the hostname with your cluster name and domain.

```
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: wildcard-gateway
  namespace: openshift-ingress
spec:
  gatewayClassName: openshift-default
  listeners:
  - allowedRoutes:
      namespaces:
        from: All
    hostname: '*.gateway.apps.rosa.rosa-s6j86.i54d.p3.openshiftapps.com'
    name: https
    port: 443
    protocol: HTTPS
    tls:
      certificateRefs:
      - name: gateway-wildcard
      mode: Terminate
```

Verify that the wildcard Gateway is functional:

```
$ oc get gateway -A
NAMESPACE           NAME               CLASS               ADDRESS                                                                  PROGRAMMED   AGE
openshift-ingress   wildcard-gateway   openshift-default   a19a0c192e2d44a6d8c50d069e46e313-777475744.us-east-2.elb.amazonaws.com   True         4m24s
$ oc get deployment -n openshift-ingress 
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
istiod-openshift-gateway             1/1     1            1           22m
router-default                       2/2     2            2           25h
wildcard-gateway-openshift-default   1/1     1            1           3m58s
```

### Expose a local test service

First create a namespace for our route:

```
$ oc new-project gateway
Now using project "gateway" on server "https://api.rosa-nnpw4.w2vo.p3.openshiftapps.com:443".
```

For testing create an example service:

$ oc new-app quay.io/rhdevelopers/echo-api:rhcl
--> Found container image 3617a52 (2 years old) from quay.io for "quay.io/rhdevelopers/echo-api:rhcl"

    * An image stream tag will be created as "echo-api:rhcl" that will track this image

--> Creating resources ...
    imagestream.image.openshift.io "echo-api" created
    deployment.apps "echo-api" created
--> Success
    Run 'oc status' to view your app.

$ oc expose deployment/echo-api --port 8080
service/echo-api exposed
```

Now create an HTTPRoute to the service to verify the gateway:

```
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo-route
  namespace: gateway
spec:
  hostnames:
  - echo.gateway.apps.rosa.rosa-s6j86.i54d.p3.openshiftapps.com
  parentRefs:
  - name: wildcard-gateway
    namespace: openshift-ingress
  rules:
  - backendRefs:
    - name: echo-api
      port: 8080
```

Once again, make sure to change the hostname to match your cluster. Curl the endpoint to ensure that the service is reachable:

```
$ curl -k https://echo.gateway.apps.rosa.rosa-s6j86.i54d.p3.openshiftapps.com/
{
  "method": "GET",
  "path": "/",
  "query_string": null,
  "body": "",
  "headers": {
    "Host": "echo.gateway.apps.rosa.rosa-s6j86.i54d.p3.openshiftapps.com",
    "User-Agent": "curl/8.15.0",
    "Accept": "*/*",
    "X-Forwarded-For": "100.64.0.3",
    "X-Forwarded-Proto": "https",
    "X-Envoy-External-Address": "100.64.0.3",
    "X-Request-Id": "383d9e66-ef46-49df-bcfe-a89e5f9580e4",
    "X-Envoy-Decorator-Operation": "echo-api.gateway.svc.cluster.local:8080/*",
    "X-Envoy-Peer-Metadata-Id": "router~10.128.0.40~wildcard-gateway-openshift-default-7ffbb96c-64mh4.openshift-ingress~openshift-ingress.svc.cluster.local",
    "X-Envoy-Peer-Metadata": "ChoKCkNMVVNURVJfSUQSDBoKS3ViZXJuZXRlcwqGAQoGTEFCRUxTEnwqegpHCh9zZXJ2aWNlLmlzdGlvLmlvL2Nhbm9uaWNhbC1uYW1lEiQaIndpbGRjYXJkLWdhdGV3YXktb3BlbnNoaWZ0LWRlZmF1bHQKLwojc2VydmljZS5pc3Rpby5pby9jYW5vbmljYWwtcmV2aXNpb24SCBoGbGF0ZXN0CjsKBE5BTUUSMxoxd2lsZGNhcmQtZ2F0ZXdheS1vcGVuc2hpZnQtZGVmYXVsdC03ZmZiYjk2Yy02NG1oNAogCglOQU1FU1BBQ0USExoRb3BlbnNoaWZ0LWluZ3Jlc3MKcAoFT1dORVISZxpla3ViZXJuZXRlczovL2FwaXMvYXBwcy92MS9uYW1lc3BhY2VzL29wZW5zaGlmdC1pbmdyZXNzL2RlcGxveW1lbnRzL3dpbGRjYXJkLWdhdGV3YXktb3BlbnNoaWZ0LWRlZmF1bHQKNQoNV09SS0xPQURfTkFNRRIkGiJ3aWxkY2FyZC1nYXRld2F5LW9wZW5zaGlmdC1kZWZhdWx0",
    "X-Envoy-Attempt-Count": "1",
    "Version": "HTTP/1.1"
  },
  "uuid": "023fe196-2473-4a5c-9e57-571186f5ed34"
}
```

### Expose remote services

Now that we've verified the Gateway API implementation on our ROSA cluster, we'll move on to how to proxy traffic to an external service.

First we need to create a external service to proxy. We'll proxy the USGS Water Data service as a simple test:

```
```

Note that we're creating a Service without a selector and then we're manually updating the EndpointSlices for that service. You could also use an "externalName" service type as well.

Verify that you can talk to USGS via the service on the echo-api pod:

```
$ oc rsh echo-api-647479f8f6-67gjf curl -k https://water-usgs.gateway.svc.cluster.local/ | head -1
<!doctype html><html itemscope itemtype=http://schema.org/WebPage lang=en class=no-js><head><meta charset=utf-8><meta name=viewport content="width=device-width,initial-scale=1,shrink-to-fit=no"><meta name=description content="USGS water data provided through web services and extensible markup language (XML) using a REST protocol."><script src=/wdfn-viz/uswds-init.min.js></script><link rel=stylesheet href=/wdfn-viz/wdfnviz-all.css><meta name=generator content="Hugo 0.128.0"><link rel=alternate type=application/rss+xml href=/index.xml><link rel=alternate type=application/json href=/index.json><link rel="shortcut icon" href=/favicons/favicon.ico><title>Water Services Web</title>
```

Once you've verified that the service is working, create an HTTPRoute to it on the Gateway:

```
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: water-usgs-route
  namespace: gateway
spec:
  hostnames:
  - water-usgs.gateway.apps.rosa.rosa-s6j86.i54d.p3.openshiftapps.com
  parentRefs:
  - name: wildcard-gateway
    namespace: openshift-ingress
  rules:
  - backendRefs:
    - name: water-usgs
      port: 443
```

Since we are proxying HTTPS traffic, we'll need a DestinationRule to specify that Envoy should initiate an HTTPS connection to the service:

```
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: water-usgs
  namespace: gateway
spec:
  host: water-usgs.gateway.svc.cluster.local
  trafficPolicy:
    tls:
      mode: SIMPLE
      sni: waterservices.usgs.gov
```

Now you should be able to curl the service from outside of the cluster:

```
$ curl -k https://water-usgs.gateway.apps.rosa.rosa-s6j86.i54d.p3.openshiftapps.com | head -1
<!doctype html><html itemscope itemtype=http://schema.org/WebPage lang=en class=no-js><head><meta charset=utf-8><meta name=viewport content="width=device-width,initial-scale=1,shrink-to-fit=no"><meta name=description content="USGS water data provided through web services and extensible markup language (XML) using a REST protocol."><script src=/wdfn-viz/uswds-init.min.js></script><link rel=stylesheet href=/wdfn-viz/wdfnviz-all.css><meta name=generator content="Hugo 0.128.0"><link rel=alternate type=application/rss+xml href=/index.xml><link rel=alternate type=application/json href=/index.json><link rel="shortcut icon" href=/favicons/favicon.ico><title>Water Services Web</title>
```

Let's proxy https://status.redhat.com. This site sits behind a Akamai, so we'll need to pass a host header.

Again, create the service:

```
---
apiVersion: v1
kind: Service
metadata:
  name: status-redhat
  namespace: gateway
spec:
  clusterIP: None
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  labels:
    kubernetes.io/service-name: status-redhat
  name: status-redhat-endpoints
  namespace: gateway
addressType: IPv4
ports:
- name: https
  port: 443
  protocol: TCP
endpoints:
- addresses:
  - 54.230.79.129
- addresses:
  - 54.230.79.21
- addresses:
  - 54.230.79.98
- addresses:
  - 54.230.79.68
```

Then, create an HTTPRoute:

```
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: status-route
  namespace: gateway
spec:
  hostnames:
  - status.gateway.apps.rosa.rosa-s6j86.i54d.p3.openshiftapps.com
  parentRefs:
  - name: wildcard-gateway
    namespace: openshift-ingress
  rules:
  - backendRefs:
    - name: status-redhat
      port: 443
    filters:
    - requestHeaderModifier:
        set:
        - name: Host
          value: status.redhat.com
      type: RequestHeaderModifier
```

Note that we're using a filter to set the Host header on requests coming through.

Here's the destination rule:

```
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: status-redhat
  namespace: gateway
spec:
  host: status-redhat.gateway.svc.cluster.local
  trafficPolicy:
    tls:
      mode: SIMPLE
      sni: status.redhat.com
```

Same as before.


```
$ curl -s -k https://status.gateway.apps.rosa.rosa-s6j86.i54d.p3.openshiftapps.com | head -10
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <!-- force IE browsers in compatibility mode to use their most aggressive rendering engine -->

    <meta charset="utf-8">
    <title>Red Hat Status</title>
    <meta name="description" content="Welcome to Red Hat&#39;s home for real-time and historical data on system performance.">

```
