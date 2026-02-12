# Container Days London 2026

## Introduction Gateway API

```bash
# Set up K3S  cluster
# Disable the by-default Traefik installation
k3d cluster create gatewayapi --port 80:80@loadbalancer --port 443:443@loadbalancer --port 8080:8080@loadbalancer --port 9090:9090@loadbalancer --k3s-arg "--disable=traefik@server:0"

# Create the namespace traefik
kubectl create namespace traefik

# Install the experimental Gateway API CRDs (not installed by default with traefik)
kubectl apply --server-side --force-conflicts -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/experimental-install.yaml

# Install Traefik
helm repo add --force-update traefik https://traefik.github.io/charts
helm upgrade --install --namespace traefik traefik traefik/traefik -f ./traefik_values.yaml

# Expose the application
kubectl apply -f ./introduction-gateway-api

curl http://whoami.docker.localhost -v
```

## Gateway API in Action

### Set up the demo

```bash
# Set up K3S  cluster
# Disable the by-default Traefik installation
k3d cluster create gatewayapi --port 80:80@loadbalancer --port 443:443@loadbalancer --port 8080:8080@loadbalancer --port 9090:9090@loadbalancer --k3s-arg "--disable=traefik@server:0"

# Create the namespace traefik
kubectl create namespace traefik

# Install the experimental Gateway API CRDs (not installed by default with traefik)
kubectl apply --server-side --force-conflicts -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/experimental-install.yaml

# Store certificates
kubectl create secret tls external-certs --namespace traefik --cert=./gateway-api-in-action/certs/external-crt.pem --key=./gateway-api-in-action/certs/external-key.pem
kubectl create secret tls internal-certs --namespace traefik --cert=./gateway-api-in-action/certs/internal-crt.pem --key=./gateway-api-in-action/certs/internal-key.pem

# Store the CA Root
kubectl create configmap internal-ca --namespace traefik --from-file ca.crt=./gateway-api-in-action/certs/rootCA.pem

# Create namespaces for the Team01
kubectl create namespace ops-team01
kubectl label namespace ops-team01 ops-team-ns=true --overwrite

kubectl create namespace dev-team01
kubectl label namespace dev-team01 dev-team-name=team01 --overwrite

# Deploy the team01 TLS certs
kubectl create secret tls external-certs --namespace ops-team01 --cert=./gateway-api-in-action/certs/external-crt.pem --key=./gateway-api-in-action/certs/external-key.pem

# Install Traefik
helm repo add --force-update traefik https://traefik.github.io/charts
helm upgrade --install --namespace traefik traefik traefik/traefik -f ./gateway-api-in-action/traefik_values.yaml
```

### Introduction

```bash
kubectl describe deployment traefik -n traefik
gwctl describe gatewayclass traefik
```

### TLSRoute demo

```bash
kubectl apply -f ./gateway-api-in-action/manifests/01-tlsroute

curl -k https://whoami-tls.docker.localhost -v

kubectl delete -f ./gateway-api-in-action/manifests/01-tlsroute
```

### BackendTLSPolicy demo

```bash
kubectl apply -f ./gateway-api-in-action/manifests/02-backendtlspolicy

curl http://whoami-tls.docker.localhost/whoami -v -L
curl -u admin:admin -L -v  http://whoami-tls.docker.localhost/whoami --location-trusted

kubectl delete -f ./gateway-api-in-action/manifests/02-backendtlspolicy
```

### ReferenceGrant demo

```bash

kubectl get ns dev-team01 -o yaml

kubectl apply -f ./gateway-api-in-action/manifests/03-referencegrant

curl https://team01.docker.localhost/ -k

kubectl delete -f ./gateway-api-in-action/manifests/03-referencegrant
```

### Migration demo

```bash
# Pre-requises
kubectl apply -f ./gateway-api-in-action/manifests/04-migration/01-basic-auth
kubectl apply -f ./gateway-api-in-action/manifests/04-migration/02-ingressclass
kubectl apply -f ./gateway-api-in-action/manifests/04-migration/03-whoami

## First step: Expose the backend with the Ingress
kubectl apply -f ./gateway-api-in-action/manifests/04-migration/04-ingress

# Reach the backend successfully every second
while true
do
    curl http://whoami-tls.docker.localhost -L  -u "user:password" --location-trusted
    sleep 1
done

## Second step: Expose the backend with a HTTPRoute

# Deploy the ingress
kubectl apply -f ./gateway-api-in-action/manifests/04-migration/05-gatewayapi

## The ingress has a bigger name than the HTTPRoute, so the priority is higher: Traefik still uses the Ingress for the routing

# Delete the ingress
kubectl delete -f ./gateway-api-in-action/manifests/04-migration/04-ingress

```
