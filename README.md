# Nephio Management Config - Approach B

This repository contains management configuration for Nephio using **Approach B** pattern.

## Key Difference from Original (Approach A)

In Approach A, networking required 4 separate PackageVariants:
1. `network-intents-my-ran.yaml`
2. `network-intents-my-core.yaml`
3. `nad-renderer-my-ran.yaml`
4. `nad-renderer-my-core.yaml`

In **Approach B**, networking requires only 2 PackageVariants:
1. `network-config-my-ran.yaml` ⭐
2. `network-config-my-core.yaml` ⭐

## Repository Structure

```
nephio-management-config-approachB/
├── cluster-contexts/
│   ├── clustercontext-my-ran.yaml
│   └── clustercontext-my-core.yaml
├── repositories/
│   ├── repository-blueprints-approachB.yaml   # Points to nephio-blueprints-approachB
│   ├── repository-my-ran.yaml
│   └── repository-my-core.yaml
└── packagevariants/
    ├── baseline/
    │   ├── baseline-my-ran.yaml
    │   └── baseline-my-core.yaml
    └── networking/
        ├── multus-my-ran.yaml
        ├── multus-my-core.yaml
        ├── whereabouts-my-ran.yaml
        ├── whereabouts-my-core.yaml
        ├── network-config-my-ran.yaml    ⭐ APPROACH B (replaces network-intents + nad-renderer)
        └── network-config-my-core.yaml   ⭐ APPROACH B (replaces network-intents + nad-renderer)
```

## Quick Setup

```bash
# Apply repositories first
kubectl apply -f repositories/

# Apply cluster contexts
kubectl apply -f cluster-contexts/

# Apply PackageVariants
kubectl apply -f packagevariants/baseline/
kubectl apply -f packagevariants/networking/
```
