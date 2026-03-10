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
                  │                     └── n3 (vpc-internet)
                  │
SMF ──── n4 (vpc-internal)
```

## Networks

| Network | Interface | Prefix | Used By |
|---------|-----------|--------|---------|
| vpc-ran | N2 | 172.2.0.0/16 | CUCP ↔ AMF |
| vpc-cu-e1 | E1 | 172.4.0.0/16 | CUCP ↔ CUUP |
| vpc-cudu-f1 | F1 | 172.5.0.0/16 | CU ↔ DU |
| vpc-internet | N3 | 172.6.0.0/16 | CUUP (user-plane) |
| vpc-internal | N4 | 172.7.0.0/16 | SMF (control) |

## Prerequisites

- Nephio management cluster with Porch, ConfigSync, IPAM controllers
- Target Kubernetes cluster registered with Nephio
- `oai-core-packages` repository registered in Porch
- `catalog-workloads-oai-ran-custom` repository registered (forked, no namespace.yaml)
- Multus CNI installed on target cluster

## Deploy Order

### Step 0: Prepare Workload Cluster

```bash
# Install CRDs (NFDeployment, NFConfig, Config, Network)
./00-prerequisites/install-crds.sh ran.kubeconfig

# Install macvlan CNI plugin on all worker nodes
ssh <node> 'bash -s' < 00-prerequisites/install-cni-plugins.sh
```

### Step 1: Apply Prerequisites (management cluster)

```bash
kubectl apply -f 00-prerequisites/oai-ran-repo.yaml
kubectl apply -f 00-prerequisites/workloadcluster.yaml
kubectl apply -f 00-prerequisites/networks.yaml
kubectl apply -f 00-prerequisites/ipprefixes.yaml
kubectl apply -f 00-prerequisites/vlanindex.yaml
```

### Step 2: Deploy Core NFs

```bash
kubectl apply -f 01-core/core-database-pv.yaml
kubectl apply -f 01-core/core-cp-operators-pv.yaml
kubectl apply -f 01-core/core-nrf-pv.yaml
kubectl apply -f 01-core/core-udr-pv.yaml
kubectl apply -f 01-core/core-udm-pv.yaml
kubectl apply -f 01-core/core-ausf-pv.yaml
kubectl apply -f 01-core/core-amf-pv.yaml
kubectl apply -f 01-core/core-smf-pv.yaml
```

### Step 3: Deploy RAN

```bash
kubectl apply -f 02-ran/oai-ran-network-pv.yaml
kubectl apply -f 02-ran/oai-ran-operator-pv.yaml
kubectl apply -f 02-ran/oai-cucp-pv.yaml
kubectl apply -f 02-ran/oai-cuup-pv.yaml
kubectl apply -f 02-ran/oai-du-pv.yaml
```

## Verify

```bash
# Check all packages are Published
kubectl get packagerevisions | grep nephio-ran

# Check pods on target cluster
kubectl --kubeconfig ran.kubeconfig get pods -A
```

## Upstream Repos

| Component | Upstream Repo | Package |
|-----------|--------------|---------|
| Core NFs | oai-core-packages | database, oai-cp-operators, oai-nrf, etc. |
| RAN NFs | catalog-workloads-oai-ran-custom | oai-ran-operator, pkg-example-cucp-bp, etc. |
| Network Blueprint | nephio-blueprints-approachb | workloads/oai-ran-network |

## Known Fixes Applied

1. **Network naming** — `vpc-internet` (N3) and `vpc-internal` (N4) match OAI upstream expectations
2. **CRDs** — Installed on workload cluster before ConfigSync syncs
3. **macvlan plugin** — Installed on worker nodes
4. **PodSecurity** — CUCP/CUUP/DU namespaces set to `privileged` in cluster-baseline
5. **Namespace conflicts** — Removed from upstream packages (forked repo)
6. **Network local-config** — Network CRs excluded from ConfigSync via `local-config: true`
