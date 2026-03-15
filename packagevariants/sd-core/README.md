# SD-Core 5G Deployment via Nephio

Deploys SD-Core 5G (free5gc-based) on a dedicated workload cluster using Nephio.

## Architecture

```
          Management Cluster (Nephio)
          ┌───────────────────────────┐
          │  Porch, IPAM, Controllers │
          └─────────┬─────────────────┘
                    │
     ┌──────────────┼──────────────┐
     │                             │
  Cluster "ran"             Cluster "core"
  (OAI-RAN)                 (SD-Core)
  ┌──────────────┐          ┌──────────────┐
  │ CUCP,CUUP,DU │          │ AMF ──N2──► gNB│
  │ OAI AMF/SMF  │          │ SMF ──N4──► UPF│
  └──────────────┘          │ UPF ──N3──► gNB│
                            │     └─N6──► inet│
                            │              │
                            │ NRF,UDR,UDM  │
                            │ AUSF,PCF,NSSF│
                            │ MongoDB,Kafka│
                            │ WebUI        │
                            └──────────────┘
```

## Networks

| Network | Interface | Prefix (aggregate) | Prefix (core /24) | Used By |
|---|---|---|---|---|
| vpc-ran | N2 | 172.2.0.0/16 | 172.2.1.0/24 | AMF ↔ gNB |
| vpc-ran | N3 | 172.3.0.0/16 | 172.3.1.0/24 | UPF ↔ gNB |
| vpc-internal | N4 | 172.1.0.0/16 | 172.1.1.0/24 | SMF ↔ UPF |
| vpc-internet | N6 | 172.0.0.0/16 | 172.0.0.0/24 | UPF → internet |

## Prerequisites

- Nephio management cluster (already running for OAI-RAN)
- Target Kubernetes cluster for SD-Core ("core") registered with Nephio
- `sdcore-packages` repo pushed to GitHub
- `nephio-core` downstream repo created on GitHub
- Multus CNI on core cluster
- macvlan CNI plugin on core cluster worker nodes
- Dynamic PV provisioning on core cluster (required by MongoDB)

## Configure Before Deploying

Update these values in the config files:

| File | Field | Update to |
|---|---|---|
| `sdcore-repos.yaml` | git.repo (downstream) | Your `nephio-core` repo URL |
| `sdcore-repos.yaml` | secretRef | Your git credentials secret name |
| `workloadcluster.yaml` | masterInterface | Your core cluster NIC name |

## Build the Operator Image

```bash
cd SD-CORE/sdcore-operators

# Build and push (update IMG to your registry)
make docker-build IMG=<your-registry>/sdcore5g-operator:v1.0.0
make docker-push IMG=<your-registry>/sdcore5g-operator:v1.0.0
```

Then update `sdcore-packages/sdcore5g-operator/operator/deployment.yaml`:
```yaml
image: <your-registry>/sdcore5g-operator:v1.0.0
```

## Deploy Order

### Step 1: Apply Prerequisites

```bash
cd SD-CORE/mgmt-config/00-prerequisites/

kubectl apply -f sdcore-repos.yaml
kubectl apply -f workloadcluster.yaml
kubectl apply -f networks-update.yaml
kubectl apply -f ipprefixes.yaml
kubectl apply -f vlanindex.yaml

# WAIT 2-3 minutes for IPAM to index routes
```

### Step 2: Deploy Control Plane

```bash
kubectl apply -f 01-cp/sdcore5g-cp-pv.yaml

# Wait for Published
kubectl get packagerevision | grep sdcore5g-cp
```

### Step 3: Deploy Operator

```bash
kubectl apply -f 02-nfs/sdcore5g-operator-pv.yaml

# Wait for Published
kubectl get packagerevision | grep sdcore5g-operator
```

### Step 4: Deploy UPF FIRST

```bash
kubectl apply -f 02-nfs/sdcore5g-upf-pv.yaml

# MUST reach Published before AMF/SMF will reconcile
kubectl get packagerevision | grep sdcore5g-upf
```

### Step 5: Deploy AMF + SMF

```bash
kubectl apply -f 02-nfs/sdcore5g-amf-pv.yaml
kubectl apply -f 02-nfs/sdcore5g-smf-pv.yaml

# These auto-wait for UPF → then reconcile
kubectl get packagerevision | grep -E "sdcore5g-(amf|smf)"
```

### Step 6: Provision Subscribers

```bash
kubectl port-forward -n sdcore5g-cp --kubeconfig core.kubeconfig \
  --address 0.0.0.0 svc/webui 5000:5000 &

# Register UE via WebUI at http://localhost:5000
```

## Verify

```bash
# All packages Published
kubectl get packagerevision | grep nephio-core

# All pods running on core cluster
kubectl --kubeconfig core.kubeconfig get pods -A
```

Expected namespaces:
- `sdcore5g-cp` — NRF, UDR, UDM, AUSF, NSSF, PCF, WebUI, MongoDB, Kafka
- `sdcore5g` — sdcore5g-operator
- `sdcore5g-upf` — UPF
- `sdcore5g-amf` — AMF (in free5gc-cp namespace)
- `sdcore5g-smf` — SMF (in free5gc-cp namespace)

## Upstream Repos

| Component | Repo | Package |
|---|---|---|
| CP NFs | sdcore-packages | sdcore5g-cp |
| Operator | sdcore-packages | sdcore5g-operator |
| AMF | sdcore-packages | pkg-example-amf-bp |
| SMF | sdcore-packages | pkg-example-smf-bp |
| UPF | sdcore-packages | pkg-example-upf-bp |
