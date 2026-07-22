---
title: "Running n8n on a Local Kubernetes Cluster: A Production-Grade Homelab Setup"
date: 2026-04-09
draft: false
tags: ["n8n", "kubernetes", "homelab", "helm", "helmfile", "k3d", "automation", "llm", "ai"]
description: "How to deploy n8n in a production-grade local Kubernetes cluster using queue mode, Helmfile, Envoy Gateway, CNPG, and Valkey — with a local LLM inference service — stood up with two commands."
ShowToc: true
---

[n8n](https://n8n.io) is a powerful open-source workflow automation tool. Running it in a real Kubernetes cluster — even locally — forces you to confront the same problems you'd face in production: secret management, queue-based execution, TLS termination, WebSocket proxying, and task runner isolation. This post walks through a fully declarative homelab setup that takes all of those seriously, and adds a local LLM inference service so that AI nodes in your workflows never leave the machine.

The full source is a small infrastructure-as-code repository. It is **not** the n8n application — it's the Helm and Kubernetes configuration that deploys it onto a local K3D cluster at `https://n8n.homelab.localhost`.

---

## Tools and Prerequisites

All tool versions are managed via [mise](https://mise.jdx.dev/), declared in `mise.toml`:

```toml
[tools]
helm      = "3"
helmfile  = "latest"
k3d       = "latest"
kubectl   = "latest"
mkcert    = "latest"
```

Run `mise trust && mise install` once to get everything. Then bring up the cluster with two tasks:

```bash
mise run env-init    # generate REDIS_PASSWORD and N8N_ENCRYPTION_KEY into .env (run once)
mise run k3d-create  # create the cluster and deploy everything
```

`env-init` writes a gitignored `.env` file with randomly generated credentials. `k3d-create` installs the mkcert CA into your system trust store, creates the cluster, and runs `helmfile sync`. Helmfile hooks handle everything else: presync hooks create namespaces and secrets, postsync hooks apply Gateway API resources — no separate manual steps.

---

## Cluster Design (`k3d.yaml`)

The cluster has 1 server node and 3 agent nodes. The server node is tainted:

```yaml
- arg: "--node-taint=CriticalAddonsOnly=true:NoExecute"
  nodeFilters:
    - server:*
```

This means all application workloads — n8n, the gateway, PostgreSQL, Valkey — are automatically scheduled onto agent nodes. The server only runs k3s built-in components (CoreDNS, local-path-provisioner, metrics-server), which already tolerate that taint. This mirrors a production pattern where control-plane nodes are reserved for infrastructure.

Storage volumes are mounted only on agent nodes:

```yaml
volumes:
  - volume: /tmp/storage:/var/lib/rancher/k3s/storage
    nodeFilters:
      - agent:*
```

Port 443 is forwarded via the k3d load balancer, so HTTPS just works from your browser.

The cluster also spins up a local Docker registry proxy (`docker-io:5000`) that caches images from Docker Hub. On a slow or metered connection this saves a lot of time when recreating the cluster.

Traefik, k3s's default ingress, is explicitly disabled — the stack uses Envoy Gateway instead.

---

## Helm Releases (`helmfile.yaml`)

Seven releases are deployed in dependency order:

| Release | Namespace | Purpose |
|---|---|---|
| `cert-manager` | `cert-manager` | TLS certificate issuance with Gateway API support |
| `envoy-gateway` | `envoy-gateway-system` | Envoy-based Gateway API implementation (v1.8.1) |
| `cnpg` | `cnpg-system` | CloudNative PostgreSQL operator |
| `n8n-cnpg` | `n8n` | PostgreSQL cluster for n8n |
| `n8n-valkey` | `n8n` | Redis-compatible queue backend |
| `n8n` | `n8n` | n8n application |

Helmfile's `needs` directive enforces ordering. Each release waits for its dependencies before installing. This matters: n8n must not start before its database and queue are ready.

Envoy Gateway is installed from its OCI chart (`docker.io/envoyproxy/gateway-helm`) as a single release — unlike some gateway implementations it doesn't require a separate CRDs chart. Importantly, it bundles its own Gateway API CRDs, so they must not be pre-installed separately: a newer CRD version in the cluster triggers a `ValidatingAdmissionPolicy` that blocks the chart's own (older) CRD installation.

The PostgreSQL cluster is provisioned via the CNPG operator's own Helm chart (`cnpg/cluster`) and the `initdb` block installs the `pgvector` extension, so AI-related workflows can store vector embeddings directly in the same database if needed.

The LLM inference operator (LLMKube) is intentionally kept out of `helmfile sync` and deployed separately:

```bash
mise run llmkube-deploy
```

This installs the LLMKube Helm chart and applies `model.yaml` in one step. Keeping it separate means the core n8n stack comes up cleanly and independently of a several-gigabyte model download.

---

## n8n in Queue Mode

Queue mode is the production execution model for n8n. Instead of the main process running workflows directly, dedicated worker pods pull jobs from a Bull queue backed by Valkey (a Redis fork). Webhook pods handle incoming HTTP triggers independently.

```yaml
worker:
  mode: queue
  allNodes: true   # DaemonSet — one worker per agent node

webhook:
  mode: queue
  allNodes: true
```

`allNodes: true` deploys both workers and webhooks as DaemonSets. Every agent node runs a worker, which means workflow execution scales horizontally with the cluster.

### External Task Runners

n8n's task runner model isolates code execution (JavaScript, Python) into separate processes. The chart supports an `external` mode where each worker pod runs a `n8n-task-runners` sidecar container.

Since chart version 1.23.x, enabling Python alongside JavaScript requires only two values:

```yaml
nodes:
  python:
    enabled: true

taskRunners:
  mode: external
  external:
    image:
      tag: "2.27.3-distroless"
```

Setting `nodes.python.enabled: true` with `taskRunners.mode: external` tells the chart to configure the sidecar for both runtimes natively — no post-renderer required. The `N8N_NATIVE_PYTHON_RUNNER: true` environment variable is also set, which is required for n8n ≥ 1.111.0 to activate Python execution. The distroless image tag is pinned explicitly to match the n8n image version and preserve the hardened image (no shell, no package manager).

The runner definitions for both languages are already baked into `/etc/n8n-task-runners.json` inside the image.

### Binary Data in Database Mode

n8n supports several binary data storage backends. The chart's `binaryData.mode` field only accepts `default`, `filesystem`, or `s3` — but `filesystem` is incompatible with queue mode (workers on different nodes can't share a local filesystem). The correct mode for this setup is `database`, which stores binary data as BLOBs in PostgreSQL.

Since the chart doesn't expose `database` as a valid enum value, it's set via an environment variable on all three pod types:

```yaml
N8N_DEFAULT_BINARY_DATA_MODE: database
```

---

## TLS and Routing

The TLS chain is:

1. **mkcert** generates a local CA and installs it into the system and browser trust stores. The CA certificate and key are stored in Secret `cert-manager/ca-cert` by the `cert-manager` presync hook.
2. **cert-manager** uses a `ClusterIssuer` backed by that secret to issue a wildcard certificate for `*.homelab.localhost`, stored in Secret `default/gateway-cert`.
3. **Envoy Gateway** terminates TLS on port 443 using that cert.
4. An **HTTPRoute** routes `n8n.homelab.localhost` to the n8n service on port 5678.

Because the mkcert CA is added to the system trust store automatically, the certificate is trusted by browsers and curl without any manual configuration.

All Gateway API resources (`GatewayClass`, `Gateway`, `ClusterIssuer`, `HTTPRoute`) are applied inline via helmfile postsync hooks — there is no separate manifest file to maintain. The `GatewayClass` in particular is not created by the Envoy Gateway Helm chart automatically and must be applied after the chart installs:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

### WebSocket Support

n8n's UI maintains a persistent WebSocket connection to `/rest/push` for real-time updates. Unlike some gateway implementations, Envoy Gateway handles WebSocket upgrades natively — no extra policy is needed to enable the protocol. Standard Gateway API `HTTPRoute` supports a `timeouts.request` field per rule; setting it to `0s` disables the per-request timeout:

```yaml
rules:
  - timeouts:
      request: 0s
    matches:
      - path:
          type: PathPrefix
          value: /
    backendRefs:
      - name: n8n
        port: 5678
```

Without this, Envoy closes the WebSocket after the default idle timeout and the UI starts showing connection errors. This replaces the Envoy-specific `BackendTrafficPolicy` CRD that was previously required for the same effect.

---

## Secret Management

Credentials are generated locally and never committed to the repository. Before creating the cluster, run:

```bash
mise run env-init
```

This writes a `.env` file (gitignored) with randomly generated values:

```
REDIS_PASSWORD=<openssl rand -hex 16>
N8N_ENCRYPTION_KEY=<openssl rand -hex 32>
```

mise loads this file automatically via `_.file = ".env"` in `mise.toml`. The helmfile presync hooks then create the Kubernetes secrets from these values — no manual `kubectl` commands are needed.

Four secrets are used at runtime:

- **`ca-cert`** (`cert-manager`): mkcert CA certificate and key. cert-manager reads this to sign the wildcard TLS certificate.
- **`valkey-auth`**: contains `redis-password`. Used by both the Valkey chart (ACL auth) and the n8n chart (`externalRedis.existingSecret`).
- **`n8n-encryption-key`**: used to encrypt stored credentials at rest. Losing this key makes all saved credentials unrecoverable — treat it like a master password.
- **`n8n-cnpg-cluster-app`**: auto-created by the CNPG operator. Contains the `password` key for the n8n database user. The n8n chart has a quirk here: it normally reads `postgres-password` from the secret, but setting `postgresql.auth.username: n8n` with `postgresql.enabled: false` tricks it into reading `password` instead, which matches what CNPG actually generates.

---

## Security Hardening

Several security measures are applied across all pod types:

| Measure | Value |
|---|---|
| Public REST API | Disabled (`api.enabled: false`) |
| Community packages | Disabled |
| SSRF protection | Enabled — blocks requests to RFC 1918 / loopback addresses from workflow nodes |
| Secure cookie | Enabled — HTTPS-only, SameSite-strict |
| Telemetry | Fully disabled (diagnostics, version notifications, template fetching, frontend hooks) |
| Task runner sidecar | Distroless image, UID 65532, read-only filesystem, no Linux capabilities |

SSRF protection is worth highlighting in a homelab context: without it, a workflow node could make HTTP requests to other services on your local network (your NAS, router admin panel, other Kubernetes services). Enabling this blocks that class of attack even from workflows you write yourself — a useful guardrail.

---

## Monitoring Readiness

The `ServiceMonitor` is disabled by default (`serviceMonitor.enabled: false` in `n8n.yaml`). To activate Prometheus metrics collection, set it to `true` and deploy `kube-prometheus-stack` into the `monitoring` namespace — no other reconfiguration of n8n is required.

---

## Local LLM Inference with LLMKube

The stack includes [LLMKube](https://github.com/defilantech/LLMKube), a Kubernetes operator that manages LLM lifecycle: it downloads the model file, provisions the inference server, and exposes a ClusterIP service — all from a pair of CRDs.

### Deploying the Model (`model.yaml`)

Two resources define the inference setup:

```yaml
apiVersion: inference.llmkube.dev/v1alpha1
kind: Model
metadata:
  name: gemma-e2b
  namespace: llmkube-system
spec:
  source: https://huggingface.co/ggml-org/gemma-4-E2B-it-GGUF/resolve/main/gemma-4-E2B-it-Q8_0.gguf
  format: gguf
  quantization: Q8_0
  resources:
    cpu: "4"
    memory: "1Gi"
---
apiVersion: inference.llmkube.dev/v1alpha1
kind: InferenceService
metadata:
  name: gemma-e2b-service
  namespace: llmkube-system
spec:
  runtime: llamacpp
  modelRef: gemma-e2b
  replicas: 1
  endpoint:
    port: 8080
    path: /v1/chat/completions
    type: ClusterIP
  resources:
    cpu: "2"
    memory: "3Gi"
```

The `Model` resource triggers a download job; the `InferenceService` then picks up the cached GGUF and launches a llama.cpp server. On first deploy this takes a while — the model is several gigabytes. Subsequent cluster recreations reuse the PVC in `llmkube-system` (10Gi, provisioned by `mise run llmkube-deploy`).

### Using the LLM in n8n Workflows

The inference service exposes an OpenAI-compatible API, so it works with n8n's built-in OpenAI credential type. Create a credential of type **OpenAI** with:

- **Base URL**: `http://gemma-e2b-service.llmkube-system.svc.cluster.local:8080/v1`
- **API Key**: any non-empty string (llama.cpp doesn't validate it)

Use the internal Kubernetes DNS name rather than the external hostname. From inside the cluster, `gemma-e2b-service.llmkube-system.svc.cluster.local` resolves directly to the ClusterIP — no round-trip through the gateway.

**SSRF caveat**: `N8N_SSRF_PROTECTION_ENABLED: true` blocks HTTP requests to RFC 1918 addresses, which includes the cluster's service CIDR. The direct service URL above will be blocked by default. To allow it, use `N8N_SSRF_ALLOWED_HOSTNAMES` in `n8n.yaml`:

```yaml
extraEnvVars:
  N8N_SSRF_ALLOWED_HOSTNAMES: gemma-e2b-service.llmkube-system.svc.cluster.local
```

The hostname allowlist takes precedence over the IP blocklist, so n8n skips the RFC 1918 check entirely for this host. The Kubernetes internal DNS zone is fully under our control, which is the intended use case for this allowlist type.

With that in place, any n8n node that accepts an OpenAI credential — AI Agent, Chat, Summarize, etc. — works against the local model. No traffic leaves the machine.

---

## Summary

This setup treats a local homelab deployment with the same seriousness as a production one. Queue mode with DaemonSet workers, encrypted secrets, SSRF protection, a proper Envoy-based gateway, isolated task runners in distroless containers — none of it is strictly necessary for a single-user local install, but all of it is good practice and costs almost nothing given a declarative approach.

The LLMKube layer adds a local inference service that slots into n8n's existing OpenAI credential system. Workflows that use AI nodes run entirely on-cluster: no API keys, no external calls, no data leaving the machine. The pgvector extension in the PostgreSQL cluster is already there if you want to add a vector store to the mix.
