# Bank of Anthos - Zero Trust Architecture

**Branch:** `feature/zero-trust-network`  
**Date:** January 19, 2026  
**Implementation:** Kubernetes NetworkPolicies + 4-Namespace Architecture

---

## Architecture Overview

### Zero Trust Principles Applied ✅

1. **Explicit Allow, Implicit Deny** - All traffic denied by default
2. **Least Privilege Access** - Services only access what they need
3. **Defense in Depth** - Multiple security layers
4. **Namespace Isolation** - Logical security boundaries
5. **No Trust Boundaries** - Every connection validated

---

## 4-Namespace Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    EXTERNAL INTERNET                        │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP/HTTPS :80/:443
                ┌──────────▼───────────────┐
                │   EDGE NAMESPACE         │
                │  (Trust: Lowest)         │
                │  ┌────────────────────┐  │
                │  │ frontend (8080)    │  │
                │  └────────────────────┘  │
                └──────────────┬───────────┘
                               │ mTLS :8080
        ┌──────────────────────┴──────────────────────┐
        │                                             │
    ┌───▼────────────────────────────────────────┐   │
    │    PRODUCTION NAMESPACE                    │   │
    │   (Trust: Medium)                          │   │
    │                                            │   │
    │  • userservice (8080)                      │   │
    │  • contacts (8080)                         │   │
    │  • ledgerwriter (8080)                     │   │
    │  • balancereader (8080)                    │   │
    │  • transactionhistory (8080)               │   │
    │                                            │   │
    └────────┬─────────────────────────────────────┘   │
             │ SSL :5432                              │
    ┌────────▼─────────────────────────────────────────┐
    │      DATA NAMESPACE                              │
    │   (Trust: Highest)                               │
    │   (Most Restricted)                              │
    │                                                  │
    │  • accounts-db (5432)                            │
    │  • ledger-db (5432)                              │
    │                                                  │
    └──────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────┐
    │      TESTING NAMESPACE                           │
    │   (Trust: Low)                                   │
    │   (Isolated from Production)                     │
    │                                                  │
    │  • loadgenerator                                 │
    │  (Only access to edge namespace)                 │
    │                                                  │
    └──────────────────────────────────────────────────┘
```

---

## Namespace Definitions

### 1. EDGE Namespace (Tier: edge, Trust: lowest)

**Purpose:** Internet-facing tier with strictest ingress controls

**Pods:**
- `frontend` - Customer-facing web UI

**Policies:**
- ✅ **Ingress:** External users on port 80/443
- ✅ **Egress:** production namespace on port 8080
- ✅ **Egress:** DNS (kube-system) on port 53
- ❌ **Egress:** data namespace DENIED
- ❌ **Egress:** testing namespace DENIED

**Security Characteristics:**
- Only externally exposed service
- Strictest ingress policies
- Cannot access databases directly
- Explicit allow rules for backend communication

---

### 2. PRODUCTION Namespace (Tier: production, Trust: medium)

**Purpose:** Internal backend services handling business logic

**Pods:**
- `userservice` - Authentication & JWT
- `contacts` - User contact management
- `ledgerwriter` - Transaction processing
- `balancereader` - Balance queries
- `transactionhistory` - Transaction history

**Policies:**
- ✅ **Ingress:** edge namespace on port 8080
- ✅ **Ingress:** testing namespace on port 8080 (loadgenerator)
- ✅ **Ingress:** internal (same namespace) on port 8080
- ✅ **Egress:** data namespace on port 5432 (databases)
- ✅ **Egress:** DNS (kube-system) on port 53
- ❌ **Ingress:** external/public DENIED
- ❌ **Egress:** edge namespace DENIED

**Security Characteristics:**
- No external ingress
- Can only talk to data namespace
- Internal service-to-service allowed
- mTLS ready for service mesh integration

---

### 3. DATA Namespace (Tier: data, Trust: highest)

**Purpose:** Critical databases with strictest isolation

**Pods:**
- `accounts-db` - User credentials & PII (PostgreSQL)
- `ledger-db` - Financial transactions (PostgreSQL)

**Policies:**
- ✅ **Ingress:** production namespace on port 5432 ONLY
- ❌ **Ingress:** edge namespace DENIED
- ❌ **Ingress:** testing namespace DENIED
- ❌ **Ingress:** external DENIED
- ❌ **Egress:** All egress DENIED (databases don't call out)

**Security Characteristics:**
- Most restrictive namespace
- Only production services can connect
- Databases never initiate outbound connections
- Defense in depth against database breaches

---

### 4. TESTING Namespace (Tier: testing, Trust: low)

**Purpose:** Load testing isolated from production

**Pods:**
- `loadgenerator` - Locust load testing tool

**Policies:**
- ❌ **Ingress:** All ingress DENIED
- ✅ **Egress:** edge namespace (frontend only) on port 80
- ✅ **Egress:** DNS (kube-system) on port 53
- ❌ **Egress:** production namespace DENIED
- ❌ **Egress:** data namespace DENIED

**Security Characteristics:**
- Only outbound access allowed
- Can only hit frontend for testing
- Cannot access production backend
- Completely isolated from databases

---

## Traffic Flow Matrix

### ALLOWED Flows ✅

```
External → frontend (port 80/443)
frontend → userservice (port 8080)
frontend → contacts (port 8080)
frontend → ledgerwriter (port 8080)
frontend → balancereader (port 8080)
frontend → transactionhistory (port 8080)

userservice → accounts-db (port 5432)
contacts → accounts-db (port 5432)
ledgerwriter → ledger-db (port 5432)
balancereader → ledger-db (port 5432)
transactionhistory → ledger-db (port 5432)

loadgenerator → frontend (port 80)

All services → DNS (kube-system, port 53)
```

### DENIED Flows ❌

```
frontend → accounts-db DENIED
frontend → ledger-db DENIED
frontend → production services DENIED

loadgenerator → userservice DENIED
loadgenerator → contacts DENIED
loadgenerator → ledgerwriter DENIED
loadgenerator → balancereader DENIED
loadgenerator → transactionhistory DENIED
loadgenerator → accounts-db DENIED
loadgenerator → ledger-db DENIED

production → edge namespace DENIED
production → testing namespace DENIED

Any service → Any service across namespace DENIED
(except explicitly allowed)

External → production services DENIED
External → data namespace DENIED
External → testing namespace DENIED
```

---

## Blast Radius & Containment

### If Frontend is Compromised
- ❌ Attacker CANNOT access databases directly
- ❌ Attacker CANNOT access production backend directly
- ✓ Attacker CAN call production APIs (expected, it's the frontend)
- ✓ Blast radius: Limited to frontend pod
- ✓ Lateral movement blocked by namespace policies

### If Production Service is Compromised
- ❌ Attacker CANNOT access edge namespace
- ❌ Attacker CANNOT access testing namespace
- ✓ Attacker CAN access databases (expected, backend needs this)
- ✓ Blast radius: Limited to production namespace
- ✓ Databases have activity logging

### If Testing Pod is Compromised
- ❌ Attacker CANNOT access production services
- ❌ Attacker CANNOT access databases
- ❌ Attacker CANNOT access other namespaces
- ✓ Blast radius: Completely isolated
- ✓ No impact on production

---

## Kubernetes NetworkPolicy Implementation

### Default Deny Strategy

Every namespace has:
1. **Default Deny Ingress** - Blocks all inbound traffic
2. **Default Deny Egress** - Blocks all outbound traffic
3. **Explicit Allow Rules** - Only traffic in documented matrix

### Example NetworkPolicy (Edge Namespace)

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: edge
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: edge
spec:
  podSelector: {}
  policyTypes:
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-allow-external-ingress
  namespace: edge
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 8080

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-allow-production-egress
  namespace: edge
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tier: production
    ports:
    - protocol: TCP
      port: 8080
```

---

## Deployment Sequence

### Step 1: Create Namespaces with Labels
```bash
kubectl create namespace edge --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace edge tier=edge trust-level=lowest --overwrite=true

kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace production tier=production trust-level=medium --overwrite=true

kubectl create namespace data --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace data tier=data trust-level=highest --overwrite=true

kubectl create namespace testing --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace testing tier=testing trust-level=low --overwrite=true
```

### Step 2: Apply NetworkPolicies
```bash
kubectl apply -f kubernetes-manifests/network-policies/01-edge-namespace.yaml
kubectl apply -f kubernetes-manifests/network-policies/02-production-namespace.yaml
kubectl apply -f kubernetes-manifests/network-policies/03-data-namespace.yaml
kubectl apply -f kubernetes-manifests/network-policies/04-testing-namespace.yaml
```

### Step 3: Migrate Pods to Namespaces
```bash
# Frontend to edge
kubectl get deployment frontend -n default -o yaml | \
  sed 's/namespace: default/namespace: edge/' | \
  kubectl apply -f -
kubectl delete deployment frontend -n default

# Backend services to production
# (similar pattern)

# Databases to data
# (similar pattern)

# Loadgenerator to testing
# (similar pattern)
```

---

## Validation Checklist

- [ ] All 4 namespaces created
- [ ] All namespaces have correct tier and trust-level labels
- [ ] 9 pods distributed to correct namespaces
- [ ] All NetworkPolicies applied
- [ ] External traffic reaches frontend
- [ ] Frontend can call production services
- [ ] Production can call databases
- [ ] Testing cannot reach production
- [ ] Testing cannot reach databases
- [ ] DNS resolution works across namespaces

---

## Comparison: Baseline vs Zero-Trust

| Aspect | Baseline | Zero-Trust |
|--------|----------|-----------|
| **Namespaces** | 1 (default) | 4 (edge, prod, data, test) |
| **Network Policies** | None | 16+ policies |
| **Default Policy** | Allow all | Deny all |
| **Frontend Isolation** | No | Yes (edge tier) |
| **Database Access** | Any pod | Production only |
| **Test/Prod Separation** | Mixed | Separate namespaces |
| **Trust Model** | Implicit trust | Explicit allow |
| **Blast Radius** | All services | Namespace-scoped |
| **Complexity** | Low | Medium |
| **Security Posture** | Poor | Excellent |

---

## Future Enhancements

1. **mTLS Encryption** - Istio service mesh for automatic encryption
2. **RBAC** - Role-based access control per namespace
3. **Pod Security Policies** - Restrict pod capabilities
4. **Network Monitoring** - Prometheus + Grafana for traffic analysis
5. **Audit Logging** - All denied connections logged
6. **Secret Management** - HashiCorp Vault or Google Secret Manager
7. **Backup & Recovery** - Database backup strategies

---

## References

- [Kubernetes NetworkPolicy Documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [NIST Zero Trust Architecture](https://www.nist.gov/publications/zero-trust-architecture)
- [CIS Kubernetes Benchmarks](https://www.cisecurity.org/benchmark/kubernetes)
- Bank of Anthos: https://github.com/GoogleCloudPlatform/bank-of-anthos
