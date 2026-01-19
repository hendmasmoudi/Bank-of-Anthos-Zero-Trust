# Bank of Anthos - Zero Trust Implementation

This repository contains two branches demonstrating the evolution of Bank of Anthos from a baseline 2-node deployment to a zero-trust security architecture with network segmentation.

## Branches

### 1. `baseline/2-node-inventory` 
**Status:** Baseline configuration
- 2-node Kubernetes cluster (Kind)
- All services in default namespace
- Asset inventory documentation
- No security implementations
- Used as reference point

### 2. `feature/zero-trust-network`
**Status:** Zero Trust Network Segmentation
- 4-namespace architecture (edge, production, data, testing)
- Frontend isolated on edge tier
- Backend services in production tier
- Databases in data tier (most restricted)
- Load testing in testing tier
- Kubernetes NetworkPolicies for zero-trust
- Deny-by-default rules with explicit allow lists

## Quick Start

### Clone and switch branches
```bash
git clone https://github.com/hendmasmoudi/Bank-of-Anthos-Zero-Trust.git
cd Bank-of-Anthos-Zero-Trust

# View baseline
git checkout baseline/2-node-inventory

# View zero-trust implementation
git checkout feature/zero-trust-network
```

### Deploy baseline
```bash
kubectl create namespace default
kubectl apply -f kubernetes-manifests/
```

### Deploy zero-trust network
```bash
cd kubernetes-manifests/network-policies
./deploy.sh
./migrate-pods.sh
```

## Architecture Comparison

| Aspect | Baseline | Zero-Trust |
|--------|----------|-----------|
| Namespaces | 1 (default) | 4 (edge, production, data, testing) |
| Network Policies | None | Deny-all + explicit allow rules |
| Frontend Isolation | No | Yes (edge tier) |
| Database Access | Any pod can access | Only production services |
| Trust Model | Trust all internal | Explicit deny, whitelist allow |

## Documentation

- [Baseline Inventory](docs/BASELINE-INVENTORY.md)
- [Zero Trust Architecture](docs/ZERO-TRUST-ARCHITECTURE.md)
- [Network Policies Guide](kubernetes-manifests/network-policies/README.md)
- [Traffic Flows](kubernetes-manifests/network-policies/TRAFFIC-FLOWS.md)

## Related

- Original Bank of Anthos: https://github.com/GoogleCloudPlatform/bank-of-anthos
- Zero Trust Principles: https://www.nist.gov/publications/zero-trust-architecture

## Author

Hend Masmoudi - Master's Thesis on Zero Trust Security in Kubernetes
