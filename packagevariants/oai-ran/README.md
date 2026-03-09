# OAI 5G RAN Deployment via Nephio

Turnkey deployment of OAI 5G Core + RAN on any Kubernetes cluster using Nephio.

## Architecture

```
         N2 (vpc-ran)         E1 (vpc-cu-e1)        F1 (vpc-cudu-f1)
AMF ◄──────────► CUCP ◄──────────────► CUUP              DU
                  │                     │                  │
                  ├── f1c ──────────────┼──── f1 ─────────┘
                  │                     │
                  │                     ├── f1u
                  │                     │
                  │                     └── n3 (n3)
                  │
SMF ──── n4 (n4)
```

## Prerequisites

- Nephio management cluster with Porch, ConfigSync, IPAM controllers
- Target Kubernetes cluster registered with Nephio
- `oai-core-packages` repository registered in Porch
- `catalog-workloads-oai-ran` repository registered in Porch
- Multus CNI installed on target cluster
- macvlan CNI plugin installed on target cluster nodes

## Deploy Order

### Step 0: Prerequisites (apply on management cluster)

```bash
kubectl apply -f 00-prerequisites/networks.yaml
kubectl apply -f 00-prerequisites/ipprefixes.yaml
kubectl apply -f 00-prerequisites/vlanindex.yaml
```

### Step 1: Core NFs (apply in order, wait for each to be Published)

```bash
kubectl apply -f 01-core/core-database-pv.yaml
kubectl apply -f 01-core/core-cp-operators-pv.yaml
# Wait for database + operators to be Published and running
kubectl apply -f 01-core/core-nrf-pv.yaml
kubectl apply -f 01-core/core-udr-pv.yaml
kubectl apply -f 01-core/core-udm-pv.yaml
kubectl apply -f 01-core/core-ausf-pv.yaml
kubectl apply -f 01-core/core-amf-pv.yaml
kubectl apply -f 01-core/core-smf-pv.yaml
```

### Step 2: RAN NFs (apply in order)

```bash
kubectl apply -f 02-ran/oai-ran-network-pv.yaml
kubectl apply -f 02-ran/oai-ran-operator-pv.yaml
# Wait for operator to be Published and running
kubectl apply -f 02-ran/oai-cucp-pv.yaml
kubectl apply -f 02-ran/oai-cuup-pv.yaml
kubectl apply -f 02-ran/oai-du-pv.yaml
```

## Verify

```bash
# Check all packages are Published
kubectl get packagerevisions | grep nephio-ran

# Check pods on target cluster
kubectl --kubeconfig <cluster>.kubeconfig get pods -A
```

## Networks

| Network | Interface | Prefix | Used By |
|---------|-----------|--------|---------|
| vpc-ran | N2 | 172.2.0.0/16 | CUCP ↔ AMF |
| vpc-cu-e1 | E1 | 172.4.0.0/16 | CUCP ↔ CUUP |
| vpc-cudu-f1 | F1 | 172.5.0.0/16 | CU ↔ DU |
| n3 | N3 | 172.6.0.0/16 | CUUP (user-plane) |
| n4 | N4 | 172.7.0.0/16 | SMF (control) |

## Upstream Repos

| Component | Upstream Repo | Package |
|-----------|--------------|---------|
| Core NFs | oai-core-packages | database, oai-cp-operators, oai-nrf, etc. |
| RAN NFs | catalog-workloads-oai-ran | oai-ran-operator, pkg-example-cucp-bp, etc. |
| Network Blueprint | catalog-workloads-oai-ran | oai-ran-network |
