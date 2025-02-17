## Overview
This repo demonstrates configuring SPIRE to issue JWT-SVID tokens for workloads, set up OIDC discovery, and configure Envoy to validate SPIRE-issued JWTs.

## Components
- **SPIRE Server**: Issues JWT-SVID tokens
- **SPIRE Agent**: Manages workload attestation
- **SPIRE OIDC Discovery**: Provides OIDC discovery endpoints and JWKS
- **PostgreSQL**: SPIRE server datastore
- **Vault**: SPIRE upstream CA 
- **Envoy**: Validates JWT tokens using SPIRE OIDC discovery

## Directory Structure
```
.
├── spire/
│   ├── configmaps/
│   │   ├── spire-server.yaml    # Server configuration
│   │   ├── spire-agent.yaml     # Agent configuration
│   │   └── spire-oidc.yaml      # OIDC discovery configuration
│   ├── deployments/
│   │   ├── spire-server.yaml
│   │   ├── spire-agent.yaml
│   │   └── spire-oidc.yaml
│   └── postgres/
│       └── docker-compose.yaml
├── istio/
│   ├── envoyfilter.yaml         # JWT validation configuration
│   └── test-service.yaml        # Test workload
```

## Installation Steps

### 1. Setup Vault

```bash
# Start Vault server
cd spire/vault
docker-compose up -d

# Initialize Vault and configure PKI
./init-vault.sh
```

### 2. Setup PostgreSQL
```bash
cd spire/postgres
docker-compose up -d
```

### 3. Deploy SPIRE and OIDC Discovery

```bash
# Create SPIRE namespace
kubectl create namespace spire

# Apply Vault certificates secret
kubectl apply -f vault-certs-secret.yaml

# Deploy SPIRE server and agent
kubectl apply -k spire

# Verify OIDC discovery endpoint
kubectl run curl-test --image=curlimages/curl -n spire-oidc -- sleep infinity
kubectl exec -n spire-oidc curl-test -- curl http://spire-oidc:8080/.well-known/openid-configuration
```

### 4. Configure Workload JWT-SVID

```bash
# Deploy test workload
kubectl apply -k workload 
```

### 5. Configure Envoy JWT Validation

```bash
# Install Istio
istioctl install -f istio-spire-config.yaml -y

# Apply EnvoyFilter for JWT validation
kubectl apply -f istio/envoyfilter.yaml
```

## Verification

### 1. Verify JWT Token Generation

```bash
# Check JWT token content
kubectl exec -n app $(kubectl get pod -n app -l app=test-workload -o jsonpath='{.items[0].metadata.name}') \
    -c main-app -- cat /run/spiffe/jwt/svid.jwt | jwt decode -

# Expected output:
Token header
------------
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "CijhattHJTvthzsb4WsoFdB5bRugGbF9"
}
Token claims
------------
{
  "aud": ["app"],
  "exp": 1739727010,
  "iat": 1739726710,
  "iss": "http://localhost:8080",
  "sub": "spiffe://example.org/ns/app/sa/test-workload"
}
```

### 2. Verify JWT Token Validation

```bash
# Verify JWT token against JWKS
kubectl exec -n app $(kubectl get pod -n app -l app=test-workload -o jsonpath='{.items[0].metadata.name}') \
    -c main-app -- cat /run/spiffe/jwt/svid.jwt | \
    step crypto jwt verify --subtle --jwks \
    <(kubectl exec -n spire-oidc curl-test -- curl -s http://spire-oidc:8080/keys)
```