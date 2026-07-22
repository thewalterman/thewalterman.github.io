---
title: "Building a Production-Grade Homelab with Kubernetes, GitOps, and Zero-Trust Security"
date: 2026-03-27
draft: false
tags: ["kubernetes", "gitops", "homelab", "security", "flux"]
description: "How to build a production-grade homelab with full OIDC-secured application stack using GitOps and zero-trust principles."
ShowToc: true
---

Running a homelab in 2026 doesn't mean a dusty server under your desk with a static IP and a self-signed certificate you have to click through every time. Modern homelabs can — and should — be built with the same rigor as production infrastructure. This is the story of exactly that: a Kubernetes homelab powered by FluxCD, OpenBao, Keycloak, and a full OpenIDConnect-secured application stack, deployed across two environments with complete GitOps automation.

The result is a platform where:

- Every change is a Git commit
- Secrets never live in YAML files
- Every app authenticates through a central identity provider
- TLS is fully automated
- Local development mirrors production as closely as possible

Let's dig into how it all works.

---

## The Big Picture

The infrastructure targets two cluster environments managed from a single Git repository:

- **k3d** — a local development cluster running in Docker, with self-signed certificates and `.homelab.localhost` domains
- **k3s** — a bare-metal/VM production cluster using Let's Encrypt certificates (via DNS-01 validation) and a real domain

[k3s](https://k3s.io) is a lightweight Kubernetes distribution — a single binary stripped of cloud-provider integrations, designed for bare-metal and edge deployments. [k3d](https://k3d.io) wraps k3s in Docker containers, making it possible to run an identical distribution locally without VMs. Because both targets run the same k3s runtime underneath, networking behavior, container lifecycle, and API compatibility are consistent — there's no separate local-dev distribution to account for.

[FluxCD](https://fluxcd.io) is the GitOps engine: it watches the repository and continuously reconciles the cluster state to match what's declared in Git. There is no `kubectl apply` in day-to-day operations. Every resource is version-controlled, reviewable, and auditable.

The repository is organized around five top-level sections:

```sh
clusters/        # Flux Kustomization entry points and FluxInstance per target
infrastructure/  # Envoy Gateway, cert-manager, External Secrets Operator
apps/            # Keycloak, OpenBao, CNPG, RustFS
monitoring/      # Prometheus, Grafana, Loki, Fluent Bit
configurations/  # Gateway, HTTPRoutes, Issuers, ExternalSecrets, PodMonitors
```

Each section follows the same overlay pattern:

```sh
<section>/
  base/   # Environment-agnostic resources (Helm releases, base configs)
  k3d/    # Patches and overrides for local development
  k3s/    # Patches and overrides for production
```

The `base/k3d/k3s` split keeps configuration DRY without branching into per-cluster repositories. `base/` holds everything environment-agnostic — the full Helm release spec, Namespace, HTTPRoute, and ExternalSecret definitions. `k3d/` and `k3s/` contain only what differs: patches for hostnames, TLS issuer references, and environment-specific Helm values. This makes diffs small and auditable — a production change is exactly one patch file, not a diff spread across duplicated configs.

A Flux `ResourceSet` in `configurations/base/` self-manages the flux-operator `HelmRelease` and `OCIRepository`, keeping the operator up to date without manual intervention.

### Local Development Fidelity

The k3d cluster is designed to mirror production topology as closely as possible without internet access. All setup tasks run through [mise](https://mise.jdx.dev), which pins tool versions and loads environment variables. Three non-obvious details make local dev work correctly:

- **mkcert CA**: `mise run k3d-create` generates a local CA with [mkcert](https://github.com/FiloSottile/mkcert), installed on the host and imported as a TLS Secret so cert-manager's k3d issuer can sign the wildcard certificate from it. Any pod that performs its own OIDC discovery call against Keycloak (Grafana, for instance) mounts that same CA into its trust store via an extra secret volume — without it, TLS verification of the self-signed issuer URL fails.
- **hostAliases**: `k3d.yaml` maps `keycloak.homelab.localhost → 172.28.0.1` (the Docker bridge gateway). Without it, pods cannot resolve the Keycloak URL used for OIDC discovery, since that hostname is only resolvable from the host.
- **Registry cache**: Docker Hub pulls are proxied through a local registry cache (`docker-io:5000`), avoiding rate limits during iterative development.

---

## Networking: Gateway API with Envoy Gateway

Rather than the classic Kubernetes `Ingress`, this setup uses the **Gateway API** — the modern, more expressive successor that offers better role separation and richer routing semantics.

[Envoy Gateway](https://gateway.envoyproxy.io/) acts as the **GatewayClass controller**, and a single `Gateway` resource handles all HTTPS traffic for the cluster. In k3d, the Gateway listens on `*.homelab.localhost` so you don't need to edit `/etc/hosts`.

Each application gets an `HTTPRoute` pointing to its backend service:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
  namespace: monitoring
spec:
  parentRefs:
    - name: gateway
      namespace: default
  hostnames:
    - grafana.homelab.localhost
  rules:
    - backendRefs:
        - name: kube-prometheus-stack-grafana
          port: 80
```

This is clean, declarative, and namespace-scoped — exactly as the Gateway API was designed

---

## Rate Limiting and Client IP Detection

Envoy Gateway's policy CRDs add HTTP-layer controls on top of the Gateway API routing model.

**BackendTrafficPolicy** applies rate limits per `HTTPRoute`. Keycloak's authentication endpoints are the obvious targets — the token endpoint (`/realms/apps/protocol/openid-connect/token`) is capped at 20 requests/minute per distinct source IP, and the login endpoint (`/realms/apps/login-actions/authenticate`) at 10 requests/minute. The policy type is `Local`, meaning limits are enforced per Envoy instance rather than shared across a fleet, which is sufficient for a single-node homelab.

For rate limiting to work correctly behind proxy, Envoy Gateway needs to know the real client IP — not proxy IP. **ClientTrafficPolicy** solves this at the Gateway level.

The reverse proxy sets `CF-Connecting-IP` to the original client IP on every proxied request and strips any client-sent version of it — so it can't be spoofed. With this policy in place, Envoy Gateway uses that header as the authoritative source IP for rate limiting, access logging, and anything else that depends on knowing who is actually making the request.

---

## TLS: Fully Automated, Zero Manual Steps

[cert-manager](https://cert-manager.io) handles certificate lifecycle in both environments. Both overlays define a `ClusterIssuer` named `ci-issuer` — same name, different type — so the base `Gateway` annotation `cert-manager.io/cluster-issuer: ci-issuer` works unchanged in both environments without patching.

**k3d (local development)**:

`mise run k3d-create` generates a local CA with mkcert and `mise run flux-bootstrap k3d` imports it as a TLS Secret in the cert-manager namespace. The k3d overlay defines a CA issuer backed by that secret. cert-manager then issues a wildcard certificate for `*.homelab.localhost`, which the Gateway uses for TLS termination. Because the same CA is trusted by both the host (mkcert installs it in the system store) and the cluster (mounted at bootstrap), browsers and in-cluster OIDC validation both work without errors.

**k3s (production)**:

The k3s overlay defines the same issuer as an ACME issuer. The DNS provider token is stored in a external vault, pulled into a Kubernetes Secret by External Secrets Operator, and passed to cert-manager for DNS-01 validation. The entire provisioning flow is automated and secret-free from a Git perspective.

---

## Secrets Management: Two Backends, One External Secrets Operator

This is the most architecturally interesting part of the stack, and where the "zero secrets in Git" principle is enforced. Both cluster targets rely on the same pattern — a `ClusterSecretStore` named `secret-store` feeding [External Secrets Operator](https://external-secrets.io) — but each points at a different backend:

```
(OpenBao on k3d | an external vault on k3s) → External Secrets Operator → Kubernetes Secrets → workload pods
```

Because the `ClusterSecretStore` name and the secret keys are identical across both targets, every `ExternalSecret` manifest in `base/` is shared unchanged between k3d and k3s — no per-target patches needed for secret consumption, only for which backend `secret-store` actually points at.

### OpenBao (k3d)

[OpenBao](https://openbao.org) is a community-maintained fork of HashiCorp Vault (following the license change). On k3d it runs in high-availability mode with Raft storage, deployed via Helm. After initialization, unseal keys are stored as a Kubernetes Secret so that a post-start hook can automatically unseal the pod on restart.

The OpenBao lifecycle is managed via `mise` tasks:

```bash
mise run bao-init     # Initialize, unseal and populate secrets
```

`secret-store` on k3d authenticates to OpenBao using the Kubernetes auth method — the `apps` role is bound to the `eso-reader` ServiceAccount:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: secret-store
spec:
  provider:
    vault:
      server: http://openbao-active.openbao:8200
      path: secrets
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: apps
```

### External Vault (k3s)

Production doesn't run OpenBao at all — it wasn't worth operating a stateful, self-managed Vault cluster on a single-node production box when a managed vault was already available. On k3s, `secret-store` instead points at that external vault through ESO's corresponding provider integration, authenticating with the compute instance's own cloud identity rather than any credential stored in the cluster:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: secret-store
spec:
  provider:
    oracle: 
      vault: ${VAULT_ID}
      region: ${VAULT_REGION}
      principalType: InstancePrincipal
```

The instance's identity is granted read access to the secret store directly through the provider's IAM — no static credentials, no in-cluster secret bootstrapping. The vault's ID and region are never committed to Git: they're injected via Flux's `postBuild.substituteFrom` on the `configurations` Kustomization, reading `${VAULT_ID}`/`${VAULT_REGION}` from a manually-created Secret in `flux-system`. Secrets themselves are seeded directly through the vault provider's own console/CLI — there's no automation task for it, deliberately, since it's a one-time setup step outside the GitOps loop.

### External Secrets Operator

Each namespace has a dedicated `ExternalSecret` resource that pulls the relevant secrets from `secret-store` and materializes them as Kubernetes Secrets. This means no application ever reads directly from OpenBao or the external vault — it just consumes a standard Kubernetes Secret. The ExternalSecret refreshes every minute, so rotation is near-real-time, on both targets.

---

## Object Storage: RustFS on k3d, an External Bucket on k3s

Object storage follows a looser version of the same split as secrets: k3d runs its own S3-compatible store in-cluster, while k3s just points at a bucket that already exists somewhere else. There's no Kubernetes-level abstraction equivalent to `secret-store` here — every consumer (CNPG's backup plugin, OpenBao's snapshot agent) already speaks the S3 API natively, so swapping backends is just a different endpoint and credential pair, not a CRD.

### RustFS (k3d)

[RustFS](https://rustfs.com) is a MinIO-API-compatible object store, deployed as a single standalone instance via Helm. It authenticates through Keycloak via OIDC — same pattern as every other app in the stack, with client credentials pulled in through an `ExternalSecret` — and its web console is reachable through its own `HTTPRoute`.

Bootstrapping the buckets is a one-shot Kubernetes `Job` that runs the `mc` client against RustFS right after it comes up: it creates the `keycloak-backup` and `openbao-backup` buckets and an admin access policy, using S3 credentials that ESO already pulled from OpenBao. It's not circular — those are static access keys, not something RustFS itself needs to back up first.

### External Bucket (k3s)

k3s doesn't run an object store workload at all — `apps/k3s/kustomization.yaml` only deploys `cnpg` and `keycloak`, nothing storage-shaped. CNPG's barman-cloud plugin talks directly to an external S3-compatible bucket instead, with the endpoint and destination path resolved by the same Flux `postBuild.substituteFrom` mechanism used for the vault connection details, reading from an `objectstore-substitutions` Secret in `flux-system`. There's no OpenBao snapshot job to worry about on this side either, since OpenBao itself doesn't run on k3s.

---

## Identity: Keycloak as the Central OIDC Provider

[Keycloak](https://www.keycloak.org) runs in its own namespace backed by a **CloudNativePG** (CNPG) managed PostgreSQL instance. It serves as the OIDC issuer for every application in the stack:

| Application | Auth mechanism |
| --- | --- |
| Grafana | `generic_oauth` (OIDC) via Helm values |
| OpenBao UI | OIDC method configured in OpenBao — k3d only |
| RustFS | OIDC login via Keycloak (credentials from ExternalSecret) — k3d only |

Role mapping is centralized: if a user belongs to the `admins` group in Keycloak, they get admin access.

Once Keycloak is ready, you can configure OIDC clients:

```bash
mise run config-oidc   # writes OIDC client secrets for Grafana and RustFS into OpenBao
```

---

## Database: CloudNativePG

[CloudNativePG](https://cloudnative-pg.io) (CNPG) is a Kubernetes operator for PostgreSQL that manages the full database lifecycle as native resources. Rather than a StatefulSet with manual replication setup, you declare a `Cluster` object and CNPG handles primary election, streaming replication, and failover automatically.

Keycloak's database is a CNPG `Cluster` defined in `apps/base/keycloak/`. Keycloak's HelmRelease has `dependsOn: cnpg`, so it never attempts to start before the database operator is ready. This ordering matters on initial bring-up — Keycloak will crash-loop if it can't reach PostgreSQL.

Backups are configured natively through the `cluster` Helm chart's `backups.*` values (cloudnative-pg/charts `cluster` >=0.8.0): setting `backups.method: plugin` makes the chart itself provision the `ObjectStore` and a daily `ScheduledBackup` against S3 via the barman-cloud plugin — no hand-written backup manifests needed alongside the cluster definition.

---

## Backup Strategy

Backups now run on **both** targets, shipping to the object store described above — RustFS on k3d, the external bucket on k3s.

- **k3d**: the OpenBao Raft-snapshot `CronJob` and Keycloak's CNPG `ScheduledBackup` both ship to buckets on the in-cluster RustFS instance (`openbao-backup` and `keycloak-backup` respectively) — useful for exercising the restore path locally without touching any external service.
- **k3s**: only Keycloak's CNPG backup runs (OpenBao doesn't exist on k3s — see Secrets Management above), shipping to an external S3-compatible bucket. The endpoint and destination path are resolved by Flux's `postBuild` variable substitution on the `apps` Kustomization, sourced from the `objectstore-substitutions` Secret, so no bucket details are hardcoded in Git.

---

## Monitoring: Prometheus, Grafana, and Logs

The `kube-prometheus-stack` Helm release deploys Prometheus, Grafana, and Alertmanager (this k3s only) in the `monitoring` namespace.

[Prometheus](https://prometheus.io) is configured to scrape all ServiceMonitors and PodMonitors in the cluster, so adding monitoring for a new app is as simple as deploying a `PodMonitor` or `ServiceMonitor` resource.

Flux itself is instrumented: dedicated PodMonitors cover all six Flux controllers (`source-controller`, `kustomize-controller`, `helm-controller`, `notification-controller`, `image-reflector-controller`, `image-automation-controller`). Custom Grafana dashboards visualize Flux reconciliation state and Kubernetes cluster health.

[Grafana](https://grafana.com/oss/grafana) authenticates through Keycloak via OIDC, and its admin credentials are pulled from OpenBao via ExternalSecret. Alert notifications from Alertmanager are wired to Flux's notification system, so Kustomization failures can trigger alerts.

### Log Aggregation: Loki + Fluent Bit

[Loki](https://grafana.com/oss/loki) runs in SingleBinary mode — one replica, filesystem persistence on a PVC. It's deliberately simple: high availability on the log store isn't a homelab requirement, and SingleBinary avoids the operational overhead of separate read/write/backend components. Logs are queryable in Grafana through LogQL via a preconfigured datasource.

**Fluent Operator** manages Fluent Bit through Kubernetes CRDs (`FluentBit`, `ClusterFilter`, `ClusterOutput`) rather than a raw ConfigMap. Log routing rules are version-controlled Kubernetes resources; adding a filter or changing the output target doesn't require touching the DaemonSet spec — Fluent Operator reconciles the Fluent Bit configuration automatically. The DaemonSet reads from containerd's log socket on each node and ships to Loki's push API. Both components expose ServiceMonitors so Prometheus scrapes their own metrics alongside application logs.

---

## Bootstrap: From Zero to Running Cluster

The `mise.toml` defines the full bootstrap sequence as runnable tasks:

```bash
# 1. Generate .env with random secrets
mise run env-init <github_user> <github_token>

# 2. Install mkcert CA and create k3d cluster
mise run k3d-create

# 3. Bootstrap Flux (installs flux-operator, creates GitHub secret, applies FluxInstance)
mise run flux-bootstrap k3d

# 4. Wait for openbao-0 Ready, then initialize OpenBao
mise run bao-init   # prints ROOT_TOKEN — save it

# 5. Wait for Keycloak Ready, then import realm
mise run import-realm

# 6. Configure OIDC on OpenBao
mise run config-oidc <ROOT_TOKEN>
```

After step 3, Flux takes over and drives the cluster toward its declared state. Steps 4–6 are the only manual interventions: OpenBao must be initialized before it can serve secrets to ESO, Keycloak must be running before the realm can be imported, and OIDC configuration requires both to be ready.

---

## Key Design Principles

Looking at this setup as a whole, a few clear principles emerge:

1. **Everything is in Git.** No out-of-band configuration. Flux reconciles continuously.
2. **Secrets never touch Git.** OpenBao (or the external vault, in production) is the source of truth; ExternalSecret bridges it to Kubernetes.
3. **Single identity provider.** Keycloak is the OIDC issuer for every application, including the API server.
4. **Local mirrors production.** k3d has the same topology as k3s — different TLS issuer, different domain, but identical GitOps structure.
5. **Automation over manual steps.** `mise` tasks encode the bootstrap sequence; image automation handles deployments.

---

## What's Running

| Component | Role |
| --- | --- |
| Flux | GitOps engine |
| Envoy Gateway | Gateway API controller, ingress |
| cert-manager | TLS automation |
| External Secrets Operator | Secret sync from OpenBao / external vault |
| OpenBao | Secrets store, OIDC auth (k3d only) |
| Keycloak | Central OIDC identity provider |
| CloudNativePG | PostgreSQL operator |
| RustFS | S3-compatible object store, OIDC login (k3d only) |
| Prometheus | Metrics collection |
| Grafana | Metrics visualization |
| Alertmanager | Alert routing (k3s only) |
| Loki | Log aggregation |
| Fluent Bit | Log collection and shipping (Fluent Operator) |

---

## Conclusion

This homelab is not particularly simple — but that's the point. It demonstrates that a solo operator can build and maintain infrastructure that would be comfortable in a professional production environment: GitOps-driven, secret-free in Git, SSO-authenticated, TLS-automated, and observable.

The combination of **FluxCD** for GitOps, **OpenBao/an external vault** for secrets, **Keycloak** for identity, and **Envoy Gateway** for networking represents a coherent, well-integrated stack where each component plays a clear role. The multi-environment Kustomize pattern keeps configuration DRY without sacrificing environment-specific flexibility.

If you're building your own homelab and want to move beyond `kubectl apply -f` and manually managed secrets, this architecture is a reference point — production thinking, homelab scale.
