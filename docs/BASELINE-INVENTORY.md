# Bank of Anthos - Baseline Inventory

**Branch:** `baseline/2-node-inventory`  
**Date:** January 19, 2026  
**Environment:** Kind Kubernetes (2 nodes + 1 control plane)  
**Namespace:** default (no segmentation)

---

## Cluster Configuration

### Infrastructure
- **Type:** Local Kubernetes (Kind - Kubernetes in Docker)
- **Nodes:** 2 worker nodes + 1 control plane
- **Network:** Docker bridge network
- **CNI:** Kindnet
- **Storage:** Local Path Provisioner

### System Components
```
kube-system namespace:
├── coredns (2 replicas)
├── etcd-boa-control-plane
├── kube-apiserver-boa-control-plane
├── kube-controller-manager-boa-control-plane
├── kube-proxy (2 replicas)
├── kube-scheduler-boa-control-plane
└── kindnet (2 replicas)

local-path-storage namespace:
└── local-path-provisioner

kube-public namespace:
└── (reserved)
```

---

## Bank of Anthos Assets

### Total Assets: 9 Microservices

#### Frontend Services (1)
| Service | Type | Language | Port | Pod Count | Status |
|---------|------|----------|------|-----------|--------|
| frontend | Web UI | Python | 8080 | 1 | Running |

#### User Management Services (2)
| Service | Type | Language | Port | Pod Count | Status |
|---------|------|----------|------|-----------|--------|
| userservice | Authentication | Python | 8080 | 1 | Running |
| contacts | Contact Management | Python | 8080 | 1 | Running |

#### Ledger/Transaction Services (3)
| Service | Type | Language | Port | Pod Count | Status |
|---------|------|----------|------|-----------|--------|
| ledgerwriter | Transaction Writer | Java | 8080 | 1 | Running |
| balancereader | Balance Cache | Java | 8080 | 1 | Running |
| transactionhistory | History Cache | Java | 8080 | 1 | Running |

#### Data Storage (2)
| Service | Type | Database | Port | Pod Count | Status |
|---------|------|----------|------|-----------|--------|
| accounts-db | Database | PostgreSQL 16 | 5432 | 1 (StatefulSet) | Running |
| ledger-db | Database | PostgreSQL 16 | 5432 | 1 (StatefulSet) | Running |

#### Load Testing (1)
| Service | Type | Language | Status |
|---------|------|----------|--------|
| loadgenerator | Load Testing | Python/Locust | Running |

---

## Current State (Baseline)

### Namespace Distribution
```
default namespace:
├── accounts-db (Pod)
├── balancereader (Deployment)
├── contacts (Deployment)
├── frontend (Deployment)
├── ledger-db (Pod)
├── ledgerwriter (Deployment)
├── loadgenerator (Deployment)
├── transactionhistory (Deployment)
└── userservice (Deployment)

Total: 9 pods in default namespace
```

### Service Network Connectivity
```
All Services are accessible to each other:
- frontend → can reach all backend services
- backend → can reach all databases
- loadgenerator → can reach all services
- No network policies enforced
- No isolation between services
```

### Port Mapping
```
External:
  frontend: LoadBalancer :80 → pod :8080

Internal ClusterIP:
  userservice: :8080
  contacts: :8080
  ledgerwriter: :8080
  balancereader: :8080
  transactionhistory: :8080
  accounts-db: :5432
  ledger-db: :5432
```

---

## Asset Criticality Assessment

### CRITICAL (Mission-Critical)
- **ledger-db** - Source of truth for all transactions
- **accounts-db** - User authentication and credentials
- **ledgerwriter** - Core transaction processing
- **userservice** - Authentication gateway

### HIGH (Important)
- **frontend** - Public-facing entry point
- **balancereader** - User balance visibility
- **transactionhistory** - Transaction history access

### MEDIUM (Supporting)
- **contacts** - User contact management
- **loadgenerator** - Load testing (non-production)

---

## Data Sensitivity

### Very High (PII + Financial)
- **ledger-db:** All financial transactions (sensitive)
- **accounts-db:** User credentials, personal information (PII)

### High (Financial Data)
- **ledgerwriter:** Transaction validation and processing
- **balancereader:** Current user balances
- **transactionhistory:** Past transaction records

### Medium (User Data)
- **userservice:** User authentication
- **contacts:** User contact information
- **frontend:** User sessions and interactions

### Low (Load Testing)
- **loadgenerator:** Synthetic test data only

---

## Current Configuration Issues

### Security Concerns (Baseline)
1. ❌ **No Network Policies** - All traffic unrestricted
2. ❌ **Single Namespace** - No isolation between services
3. ❌ **Frontend Exposed** - Same namespace as databases
4. ❌ **Test/Prod Mixed** - Loadgenerator in production namespace
5. ❌ **No Encryption** - No TLS/SSL between services
6. ❌ **No RBAC** - All pods have same access level

### Infrastructure Concerns
1. ❌ **Single Replicas** - No high availability
2. ❌ **No Backup** - Databases have no backup mechanism
3. ❌ **No Monitoring** - No observability stack
4. ❌ **No Resource Limits** - Pods can consume unlimited resources
5. ❌ **HTTP Only** - Frontend has no HTTPS

---

## Deployment Requirements

### Prerequisites
- Docker (for Kind cluster)
- kubectl 1.24+
- Skaffold 2.9+
- Git

### To Deploy This Baseline
```bash
# Clone and switch to baseline branch
git clone https://github.com/hendmasmoudi/Bank-of-Anthos-Zero-Trust.git
cd Bank-of-Anthos-Zero-Trust
git checkout baseline/2-node-inventory

# Create Kind cluster (if needed)
kind create cluster --name boa

# Deploy application
kubectl apply -f kubernetes-manifests/

# Access frontend
kubectl port-forward svc/frontend 8080:80
# Visit http://localhost:8080
```

---

## Kubernetes Resource Summary

### Deployments (6)
- frontend (1 replica)
- userservice (1 replica)
- contacts (1 replica)
- ledgerwriter (1 replica)
- balancereader (1 replica)
- transactionhistory (1 replica)
- loadgenerator (1 replica)

### StatefulSets (2)
- accounts-db (1 replica)
- ledger-db (1 replica)

### Services (9)
- frontend (LoadBalancer)
- userservice (ClusterIP)
- contacts (ClusterIP)
- ledgerwriter (ClusterIP)
- balancereader (ClusterIP)
- transactionhistory (ClusterIP)
- accounts-db (ClusterIP)
- ledger-db (ClusterIP)
- loadgenerator (N/A)

### ConfigMaps
- config.yaml (application configuration)

### Secrets
- jwt-secret (JWT signing key)

---

## Next Steps: Zero Trust Implementation

The `feature/zero-trust-network` branch implements:
1. ✅ 4-namespace architecture
2. ✅ Kubernetes NetworkPolicies
3. ✅ Deny-by-default access control
4. ✅ Frontend isolation on edge tier
5. ✅ Backend isolation in production tier
6. ✅ Database protection in data tier
7. ✅ Test isolation in testing tier

See [ZERO-TRUST-ARCHITECTURE.md](../docs/ZERO-TRUST-ARCHITECTURE.md) for details.

---

## References

- [Bank of Anthos README](../../bank-of-anthos/README.md)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kind Documentation](https://kind.sigs.k8s.io/)
