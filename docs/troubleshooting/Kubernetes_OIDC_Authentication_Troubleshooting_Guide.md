# Kubernetes OIDC Authentication Failure -- Troubleshooting Guide

## Overview

This guide provides a generic troubleshooting workflow for Kubernetes
clusters using OIDC authentication (Dex or similar providers). It is
written without environment-specific details so it can be shared
publicly.

## Symptoms

Typical error:

``` text
error: You must be logged in to the server (Unauthorized)
```

API server logs commonly show errors such as:

``` text
failed to verify signature
fetching keys
context deadline exceeded
```

or

``` text
failed to fetch JWKS
connection refused
```

These usually indicate that the Kubernetes API Server cannot reach the
OIDC issuer's JWKS endpoint.

## Troubleshooting Workflow

### 1. Verify the Identity Provider

``` bash
kubectl get pods -n <oidc-namespace>
kubectl get svc -n <oidc-namespace>
```

### 2. Verify the Identity Provider Internally

``` bash
kubectl debug node/<node> -it --image=busybox
```

``` bash
wget -S -O- http://<oidc-service>:<port>/keys
```

Expected result: HTTP 200 with a JSON Web Key Set (JWKS).

### 3. Verify External Access

``` bash
curl -vk https://<oidc-domain>/.well-known/openid-configuration
curl -vk https://<oidc-domain>/keys
```

If these intermittently fail, investigate ingress or networking.

### 4. Verify Load Balancer and Ingress

``` bash
kubectl describe svc <ingress-service>
kubectl get endpoints <ingress-service>
kubectl get endpointslices
kubectl get pods -n <ingress-namespace> -o wide
```

### 5. Test From Different Nodes

Test the OIDC endpoint from both a worker node and a control plane node.

If workers succeed but the control plane cannot reach the endpoint, the
issue is likely between the API Server and the ingress VIP.

### 6. Check API Server Logs

``` bash
kubectl logs -n kube-system <kube-apiserver-pod>
```

Look for authentication, JWKS, issuer or timeout errors.

### 7. Validate OIDC Configuration

Verify:

-   Issuer URL
-   Client ID
-   Client Secret
-   Redirect URI

Generate a token manually:

``` bash
kubectl oidc-login get-token   --oidc-issuer-url=https://<issuer>   --oidc-client-id=<client-id>   --oidc-client-secret=<client-secret>
```

If token generation succeeds, the IdP configuration is probably correct.

## Recovery Procedure

Restart networking components one at a time, validating after each
restart.

``` bash
kubectl rollout restart daemonset metallb-speaker -n metallb-system
```

``` bash
kubectl rollout restart daemonset kube-proxy -n kube-system
```

``` bash
kubectl rollout restart daemonset <cni-daemonset> -n kube-system
```

``` bash
kubectl rollout restart deployment <ingress-controller> -n <ingress-namespace>
```

## Validation

``` bash
curl https://<oidc-domain>/.well-known/openid-configuration
kubectl oidc-login get-token
kubectl get ns
```

Ensure API Server authentication errors are no longer present.

## Root Cause

In this case, the identity provider, services, and LoadBalancer were
healthy. The issue was stale networking between the Kubernetes API
Server and the ingress endpoint hosting the OIDC provider. Restarting
the ingress controller restored connectivity to the JWKS endpoint,
allowing token validation to succeed.

## Key Takeaways

-   Keep a break-glass administrator kubeconfig.
-   Always test connectivity from the control plane.
-   A healthy IdP does not guarantee the API Server can reach it.
-   API Server logs usually identify the failure quickly.
-   Restart networking components one at a time to isolate the failing
    layer.
