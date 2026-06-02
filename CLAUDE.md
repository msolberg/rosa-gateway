# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository demonstrates how to use the Gateway API on a ROSA (Red Hat OpenShift Service on AWS) cluster to provide access to external web services. It contains OpenShift/Kubernetes manifests only—no application code to build or compile.

The Gateway API implementation uses:
- OpenShift Ingress Operator as the controller
- Istio for the Gateway implementation
- Envoy Proxy for traffic routing

## Architecture

### Resource Hierarchy

1. **GatewayClass** (`openshift-default.yaml`): Defines the controller type
   - **Must** be named `openshift-default` - the Ingress Operator only manages this specific name
   - Managed by `openshift.io/gateway-controller/v1`

2. **Gateway** (`wildcard-gateway.yaml`): Provisions the actual gateway infrastructure
   - Deployed in `openshift-ingress` namespace
   - References the GatewayClass
   - Requires TLS certificates stored as a secret in the same namespace
   - Creates an Istio deployment (`istiod-openshift-gateway`) and gateway deployment

3. **HTTPRoute** (various `*-route.yaml` files): Route traffic from the Gateway to backend services
   - Can be deployed in any namespace (examples use `gateway` namespace)
   - References the Gateway using `parentRefs`
   - Specifies hostnames and backend services

### External Service Proxy Pattern

To proxy external services through the Gateway, three resources are typically needed:

1. **Headless Service**: `clusterIP: None` allows Envoy to see individual endpoint addresses
2. **EndpointSlice**: Defines the external IP addresses to proxy to
3. **DestinationRule** (for HTTPS backends): Istio resource to configure TLS origination
   - Uses `mode: SIMPLE` to enable TLS
   - Sets SNI hostname to match the external service's certificate

Examples:
- `status-redhat-service.yaml` + `status-redhat-destination-rule.yaml` (HTTPS)
- `external-service.yaml` (HTTP)
- `water-usgs-service.yaml` + `water-usgs-destination-rule.yaml` (HTTPS)

## Common Commands

### Apply Manifests

```bash
# Apply individual resources
oc create -f openshift/gateway/base/openshift-default.yaml
oc create -f openshift/gateway/base/wildcard-gateway.yaml
oc apply -f openshift/gateway/base/echo-route.yaml

# Apply using Kustomize
oc apply -k openshift/gateway/base/
```

### Verify Gateway API Status

```bash
# Check GatewayClass is accepted
oc get gatewayclass openshift-default -o yaml

# Check Gateway status and address
oc get gateway -A
oc get gateway wildcard-gateway -n openshift-ingress -o yaml

# Check HTTPRoutes
oc get httproute -A

# Verify Istio controller is running
oc get pods -n openshift-ingress
oc get deployment -n openshift-ingress
```

### Certificate Management

The Gateway requires TLS certificates for the wildcard domain:

```bash
# Extract default ingress certificates
oc get secret/default-ingress-cert -n openshift-ingress -o yaml

# Create secret for Gateway (after extracting cert and key)
oc -n openshift-ingress create secret tls gwapi-wildcard --cert=default.cert --key=default.key
```

## File Structure

- `openshift/gateway/base/`: All Gateway API manifests
  - `kustomization.yaml`: Kustomize configuration
  - `namespace.yaml`: Gateway namespace definition
  - `openshift-default.yaml`: GatewayClass definition
  - `wildcard-gateway.yaml`: Gateway definition (requires hostname update for specific cluster)
  - `echo-route.yaml`: Example HTTPRoute for internal service
  - `*-service.yaml`: External service proxy definitions (headless Service + EndpointSlice)
  - `*-destination-rule.yaml`: Istio TLS configuration for HTTPS backends
  - `*-route.yaml`: HTTPRoute definitions for each proxied service

## Important Notes

- The `wildcard-gateway.yaml` contains a cluster-specific hostname that must be updated for your ROSA cluster's ingress domain
- GatewayClass name is fixed—do not rename `openshift-default`
- Gateway must be in `openshift-ingress` namespace
- HTTPRoutes reference the Gateway across namespaces using `parentRefs`
- For HTTPS external services, always include a DestinationRule to configure TLS origination
