---
title: "Managing Multiple Kubernetes Clusters with FluxCD and a Hub-Spoke Model"
date: 2026-03-30
draft: false
tags: ["kubernetes", "gitops", "flux", "multi-cluster", "hub-spoke"]
description: "How to manage multiple Kubernetes clusters from a single Git repository using FluxCD and a hub-spoke topology."
ShowToc: true
---

Running a single Kubernetes cluster is hard enough. Running several — keeping them consistent, deploying to them safely, and not losing your mind in the process — is where things get interesting. This post walks through a GitOps setup that manages staging and production Kubernetes clusters from a single Git repository using FluxCD and a hub-spoke topology.

---

## The Problem with Multi-Cluster Management

Most teams start with one cluster. Then they add a staging environment. Then a production environment in a different region. Before long they have a lot of `kubectl apply` scripts, half-baked CI pipelines, and a nagging feeling that staging and production have quietly diverged.

The standard GitOps answer is: Git is the source of truth, and a controller reconciles the cluster toward that truth continuously. But when you have *multiple* clusters, a new question arises: where does the controller run?

One option is to run Flux on every cluster independently. That works, but it means managing Flux itself across every cluster, and there's no single place to observe what's happening across your fleet.

A cleaner option is the **hub-spoke model**: one dedicated hub cluster runs Flux, and it is responsible for reconciling manifests into all the remote clusters.

---

## Architecture

```sh
GitHub (source of truth)
  └─ Flux on hub cluster
       ├─ hub/production.yaml
       └─ hub/staging.yaml
```

There are three Kubernetes clusters in total:

- **Hub** — runs Flux and nothing else of substance. It is the control plane for the fleet.
- **Staging** — receives workloads reconciled by the hub.
- **Production** — same, but production.

---

## Bootstrapping Flux with flux-operator

Rather than using `flux bootstrap github` (which commits generated manifests back to the repository), the hub cluster is bootstrapped with [flux-operator](https://fluxoperator.dev/). The operator is installed once via Helm, and a single `FluxInstance` custom resource declares the desired Flux distribution and the GitRepository sync configuration. The operator owns the lifecycle of the Flux controllers from that point on.

This keeps the repository clean — there are no auto-generated `gotk-components.yaml` files committed by a bootstrap command — and upgrades to Flux itself become a one-line change to `spec.distribution.version` in the `FluxInstance`.

---

## How Flux Reaches the Remote Clusters

Flux on the hub needs credentials to talk to the remote cluster API servers. These are stored as Kubernetes secrets on the hub itself — one kubeconfig secret per remote cluster, in a dedicated namespace:

```sh
hub cluster
  ├── namespace: staging
  │     └── secret: cluster-kubeconfig
  └── namespace: production
        └── secret: cluster-kubeconfig
```

Each Flux `Kustomization` resource on the hub references its remote cluster's kubeconfig via `spec.kubeConfig.secretRef`. When Flux reconciles that Kustomization, it applies the manifests to the remote cluster instead of the hub.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: tenants
  namespace: staging
spec:
  kubeConfig:
    secretRef:
      name: cluster-kubeconfig
  path: ./clusters/staging/tenants
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
```

---

## Deployment Stages and Dependencies

Each remote cluster receives four stages of configuration. The dependency chain is: tenants → infrastructure → {configurations, apps in parallel}. Flux waits for each stage to be ready before starting anything that depends on it.

**1. Tenants** — Creates the namespace, a `flux-cluster-admin` ServiceAccount, and a ClusterRoleBinding granting it cluster-admin. This is the identity that Flux will impersonate when applying subsequent stages to the remote cluster.

**2. Infrastructure** — Deploys [Envoy Gateway](https://gateway.envoyproxy.io/) via HelmRelease. Because Helm needs to talk to the remote cluster, each HelmRelease is patched with `spec.kubeConfig` and `spec.serviceAccountName` at reconciliation time.

**3. Configurations** — Creates the `GatewayClass`, `Gateway`, and `HTTPRoute` resources (Kubernetes Gateway API) needed to route traffic into the cluster.

**4. Apps** — Deploys the apps as a HelmRelease, with environment-specific values patched per cluster.

The dependency graph ensures that you never run out of apps deployed before the infrastructure is ready.

---

## Keeping Staging and Production DRY

The `deploy/` directory holds shared base definitions — the infrastructure HelmReleases, the tenant RBAC templates, the apps HelmRepository and base HelmRelease. `clusters/staging/` and `clusters/production/` are Kustomize overlays that reference these bases and apply environment-specific patches.

For example, the base definition of the app [homepage](https://gethomepage.dev/) lives in `deploy/apps/homepage.yaml`. The staging overlay at `clusters/staging/apps/homepage.yaml` patches in staging-specific values:

```yaml
spec:
  values:
    env:
      - name: HOMEPAGE_ALLOWED_HOSTS
        value: "staging.multicluster.localhost"
    config:
      widgets:
        - greeting:
            text: Staging
```

The hub-level Kustomization for staging then patches *all* HelmReleases in that path with the remote kubeconfig reference, so you don't have to repeat it per app:

```yaml
patches:
  - target:
      kind: HelmRelease
    patch: |
      - op: add
        path: /spec/kubeConfig
        value:
          secretRef:
            name: cluster-kubeconfig
      - op: add
        path: /spec/serviceAccountName
        value: flux-cluster-admin
```

This layered approach — base definitions, cluster overlays, hub-level patches — keeps the YAML minimal and avoids copy-paste between environments.

---

## What This Gets You

The hub-spoke model with Flux gives you a few things that are hard to achieve otherwise:

- **Single pane of glass** — all reconciliation events are visible from the hub. One `kubectl --context k3d-hub get kustomizations -A` shows the state of every cluster.
- **Consistent tooling** — staging and production use the same Flux version and the same Kustomize bases. Differences are explicit patches, not undocumented drift.
- **Git as the audit log** — every change to any cluster passes through a pull request. The diff is exactly what will be applied.
- **Dependency ordering** — Flux's `dependsOn` ensures clusters bootstrap in the right order every time, whether that's first setup or recovering from a failure.

For teams managing more than one cluster, this pattern scales naturally: adding a new environment means adding a new overlay directory and a new hub Kustomization, without touching any existing configuration.
