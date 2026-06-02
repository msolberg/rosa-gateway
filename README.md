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
oc create -f openshift/gateway/base/openshift-default.yaml
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
$ oc get secret/default-ingress-cert -n openshift-ingress -o yaml
# echo the certifiate and key to base64 -d into two files
# then create the secret:
$ oc -n openshift-ingress create secret tls gwapi-wildcard --cert=/tmp/default.cert --key=/tmp/default.key 
secret/gwapi-wildcard created
```

### Gateway

Finally, create a wildcard Gateway object:

TODO: fix domain

```
$ oc create -f openshift/gateway/base/wildcard-gateway.yaml 
gateway.gateway.networking.k8s.io/wildcard-gateway created
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

### HTTPRoute

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

```
$ oc apply -f openshift/gateway/base/echo-route.yaml 
```
