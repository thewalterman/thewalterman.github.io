---
title: "Self-Hosting GitLab CE on Kubernetes with Helmfile"
date: 2026-06-21
draft: false
tags: ["kubernetes", "gitlab", "helmfile", "envoy-gateway", "cloudnative-pg", "rustfs"]
description: "A production-grade GitLab CE deployment on Kubernetes using Helmfile, Envoy Gateway, CloudNative-PG, and RustFS."
ShowToc: true
---

Self-hosting GitLab is a common enough idea. Actually running it well — with proper TLS, a real database operator, S3-compatible object storage, and secrets management that doesn't involve pasting credentials into YAML — is where most guides stop short. This post walks through a Helmfile-based deployment of GitLab CE on Kubernetes that takes those pieces seriously.

---

## Why Helmfile

The GitLab Helm chart has a lot of moving parts: it depends on an external PostgreSQL database, an external Redis-compatible cache, and an S3-compatible object store. Managing those dependencies — and making sure they exist before the GitLab chart tries to use them — is exactly what Helmfile is good at.

Helmfile orchestrates multiple Helm releases in dependency order. Each release declares its `needs`, and Helmfile ensures that dependencies are deployed and ready before moving to anything that depends on them. For a stack like this, that ordering matters a lot.

```
envoy-gateway ──┐
cert-manager ───┤
cnpg ──────────►cnpg-gitlab ───┐
                               │
                valkey ────────┼──► gitlab
                               │
                rustfs ────────┘
```

---

## Ingress: Envoy Gateway and the Gateway API

Rather than nginx-ingress, this setup uses **Envoy Gateway** as the Gateway API controller. The Gateway API is the modern successor to Kubernetes `Ingress` — more expressive, better role separation, and where the ecosystem is clearly heading.

The `GatewayClass` and `Gateway` are both created in a `postsync` hook immediately after Envoy Gateway is installed — before cert-manager runs. This ordering is intentional: cert-manager needs the Gateway to exist in order to perform the HTTP-01 ACME challenge.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: gitlab-gw
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gitlab-gw
  namespace: gitlab
spec:
  gatewayClassName: gitlab-gw
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
    - name: gitlab-web
      hostname: gitlab.thewalterman.cloud
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: gitlab-wildcard-tls
            kind: Secret
      allowedRoutes:
        namespaces:
          from: Same
    - name: registry-web
      hostname: registry.thewalterman.cloud
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: gitlab-wildcard-tls
            kind: Secret
      allowedRoutes:
        namespaces:
          from: Same
```

The listener names (`gitlab-web`, `registry-web`) match the `sectionName` values that the GitLab chart uses in its HTTPRoute `parentRefs` — this is a non-obvious requirement.

The GitLab chart is configured with `global.gatewayApi.gatewayRef` pointing to this Gateway. With that set, the chart skips creating its own Gateway resource and only creates HTTPRoutes. Without it, Helm would try to claim ownership of an existing resource it didn't create and fail.

RustFS is internal-only and not exposed through the Gateway.

---

## TLS: cert-manager and Let's Encrypt

[cert-manager](https://cert-manager.io) manages TLS. For production, a `ClusterIssuer` is configured to use Let's Encrypt ACME with the HTTP-01 challenge type via cert-manager's `gatewayHTTPRoute` solver. The solver creates a temporary HTTPRoute on the `gitlab-gw` Gateway for each challenge — which is why the Gateway must exist before cert-manager runs.

```yaml
solvers:
  - http01:
      gatewayHTTPRoute:
        parentRefs:
          - name: gitlab-gw
            namespace: gitlab
            kind: Gateway
            group: gateway.networking.k8s.io
```

Rather than a wildcard certificate, the cert lists explicit SANs: `gitlab.thewalterman.cloud` and `registry.thewalterman.cloud`. HTTP-01 cannot validate wildcard domains, so each hostname must be listed explicitly.

The cert-manager `postsync` hook creates the `ClusterIssuer` and `Certificate` resources, then waits for the certificate to become `Ready` before returning. This means the TLS secret exists before the GitLab chart installs.


---

## Database: CloudNative-PG

The GitLab chart does not bundle PostgreSQL — it expects an external database. [CloudNative-PG](https://cloudnative-pg.io) (CNPG) runs as an operator in `cnpg-system` and manages PostgreSQL as a native Kubernetes resource.

The `cnpg-gitlab` release deploys a `Cluster` object. CNPG handles the full lifecycle: primary election, connection pooling, and health checks. GitLab connects via `gitlab-rails-db-rw.gitlab.svc.cluster.local` — the read-write service that CNPG keeps pointed at the current primary.

The Helmfile dependency chain ensures CNPG is operator-ready before the cluster object is applied, and the cluster is healthy before GitLab starts.

---

## Database Backups

CNPG's backup integration is configured entirely through Helm values. The `cnpg-gitlab` release enables continuous WAL archiving and daily base backups via barman, targeting the internal RustFS endpoint:

```yaml
backups:
  enabled: true
  provider: s3
  s3:
    bucket: cnpg-backup
    region: us-east-1
  endpointURL: http://rustfs-svc.gitlab.svc.cluster.local:9000
  secret:
    create: false
    name: cnpg-backup-s3
  wal:
    compression: gzip
  scheduledBackups:
    - name: daily-backup
      schedule: "0 0 2 * * *"
      backupOwnerReference: self
  retentionPolicy: "7d"
```

The backup credentials (`cnpg-backup-s3`) are created by a `presync` hook on the `cnpg-gitlab` release, reading from the same `.env` variables as the rest of the stack. `secret.create: false` tells the chart not to try to create it — the hook already has.

This gives you WAL archiving from the moment the cluster starts, plus a daily base backup, with 7-day retention — all without any manual configuration after `helmfile sync`.

---

## Cache: Valkey

[Valkey](https://valkey.io) is the Redis-compatible cache that GitLab uses for sessions, sidekiq queues, and cache data. It's a community fork of Redis maintained under an open-source license — functionally identical from GitLab's perspective.

The password secret is created by a Helmfile `presync` hook before the chart installs:

```bash
kubectl create secret generic valkey-auth \
  --namespace gitlab \
  --from-literal=default="${VALKEY_PASSWORD:?}" \
  --dry-run=client -o yaml | kubectl apply -f -
```

The `${VALKEY_PASSWORD:?}` syntax fails loudly if the variable is unset, catching the common mistake of running `helmfile sync` without first sourcing `.env`. The password is never written to any file tracked by Git — it lives in `.env` (gitignored) and gets materialized as a Kubernetes Secret at deploy time.

---

## Object Storage: RustFS

The GitLab chart normally ships with MinIO, but this setup replaces it with [RustFS](https://rustfs.com) — a MinIO-compatible S3 implementation. It speaks the same S3 API, which means the GitLab chart configuration is unchanged; you just point it at a different endpoint.

Credentials are never inlined in the values file. Instead, a `presync` hook creates the `rustfs-auth` Kubernetes Secret from shell environment variables, and the chart references it via `secret.existingSecret`:

```yaml
secret:
  existingSecret: rustfs-auth
```

The same hook also creates the GitLab-facing secrets (`gitlab-object-storage`, `gitlab-registry-storage`) that the GitLab chart needs to connect to RustFS.

Bucket provisioning is handled by a Kubernetes `Job` in the chart's `extraManifests`. It runs on install and upgrade, creating every bucket GitLab expects — artifacts, uploads, lfs-objects, packages, registry, and others, including the `cnpg-backup` bucket used for database backups. The Job reads credentials from `rustfs-auth` via `envFrom`, so no credentials appear in the Job spec either.

RustFS is only reachable inside the cluster. GitLab and CNPG access it via `http://rustfs-svc.gitlab.svc.cluster.local:9000` — there is no external Gateway listener for S3.

---

## Deployment Workflow

First-time setup:

```bash
mise trust && mise install          # install helm, helmfile, k3d, kubectl
mise run env-init                   # generate .env with random credentials
k3d cluster create -c k3d.yaml     # create local cluster
source .env && helmfile sync        # deploy full stack
```

Day-to-day:

```bash
source .env && helmfile diff        # preview changes against live cluster
source .env && helmfile sync        # apply all releases
```

The `source .env` is necessary because helmfile hooks read credentials from shell environment variables. Forgetting it surfaces as an explicit error (`${VAR:?}` fails) rather than a silent misconfiguration.

---

## What's Running

After a successful deploy, the following are accessible with valid TLS:

| Service | URL |
| ------- | --- |
| GitLab | `https://gitlab.thewalterman.cloud` |
| Container Registry | `https://registry.thewalterman.cloud` |

---

## What This Is Good For

The obvious use case is developing and testing GitLab CI pipelines locally — you get a real GitLab with a real runner, real artifact storage, and a container registry without pointing pipelines at a shared environment.

It's also a useful reference for anyone deploying GitLab on Kubernetes outside of the managed cloud offerings: the Helmfile dependency model, the pattern of using presync hooks for secrets (never in committed YAML), the RustFS replacement for MinIO, and the CNPG backup integration are all directly transferable to production deployments.

The full setup fits on a single k3d node with reasonable resource usage, making it practical to run alongside other development work.
