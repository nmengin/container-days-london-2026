## Set up the demo

```bash
# 1. Install K3S / K3D
# 2. Generate your external and internal certificates (you can use mkcert) and put them in the folder certs
# 3. cURL to test the connection

# Set up K3S  cluster
# Disable the by-default Traefik installation
k3d cluster create gatewayapi --port 80:80@loadbalancer --port 443:443@loadbalancer --port 8080:8080@loadbalancer --port 9090:9090@loadbalancer --k3s-arg "--disable=traefik@server:0"

# Create the namespace traefik
kubectl create namespace traefik

# Install the experimental Gateway API CRDs (not installed by default with traefik)
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/experimental-install.yaml

# Store certificates
kubectl create secret tls external-certs --namespace traefik --cert=./certs/external-crt.pem --key=./certs/external-key.pem
kubectl create secret tls internal-certs --namespace traefik --cert=./certs/internal-crt.pem --key=./certs/internal-key.pem

# Create a namespace for the Team01
kubectl create namespace team01
kubectl label namespace team01 type=approved-ns --overwrite

kubectl create secret tls external-certs --namespace team01 --cert=./certs/external-crt.pem --key=./certs/external-key.pem

# Store the CA Root
kubectl create configmap internal-ca --namespace traefik --from-file ca.crt=./certs/rootCA.pem

# Install Traefik
helm repo add --force-update traefik https://traefik.github.io/charts
helm upgrade --install --namespace traefik traefik traefik/traefik -f ./traefik_values.yaml
```

## TLSRoute demo

```bash
kubectl apply -f ./manifests/tlsroute

curl -k https://whoami-tls.docker.localhost -v

kubectl delete -f ./manifests/tlsroute
```

## BackendTLSPolicy demo

```bash
kubectl apply -f ./manifests/backendtlspolicy

curl http://whoami-tls.docker.localhost/whoami -v
curl -u admin:admin -L -v  http://whoami-tls.docker.localhost/whoami

kubectl delete -f ./manifests/backendtlspolicy
```

## ReferenceGrant demo

```bash
kubectl apply -f ./manifests/referencegrant

curl https://whoami-tls.docker.localhost/ -v -k

kubectl delete -f ./manifests/referencegrant
```