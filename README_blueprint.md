# Nephio Blueprints - Approach B

This repository contains Nephio blueprints using **Approach B** pattern for NAD generation.

## Key Difference from Original (Approach A)

| Aspect | Approach A (Original) | Approach B (This Repo) |
|--------|----------------------|------------------------|
| NetworkIntents | Separate `network-intents` package | Embedded in `network-config` package |
| NAD generation | Relies on Porch injector (broken) | nad-fn processes embedded intents |
| Packages needed | 2 (network-intents + nad-renderer) | 1 (network-config) |
| Result | ❌ NADs not generated | ✅ NADs generated correctly |

## Repository Structure

```
nephio-blueprints-approachB/
├── cluster-baseline/              # Base cluster configuration
│   ├── Kptfile
│   ├── namespaces.yaml
│   ├── pod-security.yaml
│   └── storage-class.yaml
└── networking/
    ├── network-config/            # ⭐ APPROACH B - All-in-one NAD package
    │   ├── Kptfile                # Contains nad-fn in pipeline
    │   ├── network-intents.yaml   # EMBEDDED NetworkIntents
    │   └── nad-renderer-config.yaml
    ├── multus-cni/                # Multus CNI DaemonSet
    │   ├── Kptfile
    │   ├── nad-crd.yaml
    │   └── multus-daemonset.yaml
    └── whereabouts-ipam/          # Whereabouts IPAM
        ├── Kptfile
        └── whereabouts.yaml
```

## Why Approach B Works

The `network-config` package contains:
1. **NetworkIntents** (embedded, not injected)
2. **NADRendererConfig** 
3. **nad-fn in pipeline**

When Porch processes this package:
1. `apply-setters` applies cluster-specific values
2. `nad-fn` reads NetworkIntents **from the same package**
3. `nad-fn` generates NADs

No injection needed → No silent failure → NADs created!

## Quick Reference

Use with `nephio-management-config-approachB` repository for PackageVariants.
