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
| --- | --- | --- |
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

Queue mode is the production execution model for n8n. Instead of the main process running workflows directly, dedicated worker pods pull jobs from a Bull queue backed by Valkey (a Redis fork). A separate webhook-processor deployment handles incoming HTTP triggers independently, so a burst of webhook traffic never competes with editor/API load on the main pod.

```yaml
queueMode:
  enabled: true
  workerReplicaCount: 3

webhookProcessor:
  enabled: true
  replicaCount: 3
  disableProductionWebhooksOnMainProcess: true
```

Both replica counts are set to 3, matching the cluster's three agent nodes. The chart deploys workers and webhook processors as plain `Deployment`s with a fixed replica count rather than pinning one pod per node — there's no DaemonSet mode — so this is a sizing choice rather than an enforced topology. `disableProductionWebhooksOnMainProcess` stops the main pod from also handling production webhook traffic once the dedicated processor pods take over.

### External Task Runners

n8n's task runner model isolates code execution (JavaScript, Python) into separate processes. The chart runs a `task-runner` sidecar alongside the main and worker containers — not the webhook-processor, which never executes Code nodes:

```yaml
taskRunners:
  enabled: true
  nativePythonRunner: true
  image:
    repository: n8nio/runners
    tag: "2.27.3-distroless"
```

`nativePythonRunner: true` is the default and sets `N8N_NATIVE_PYTHON_RUNNER` automatically on every pod that gets the sidecar — no post-renderer or custom launcher config needed for either language. The distroless image tag is pinned explicitly to match the n8n image version and preserve the hardened image (no shell, no package manager).

The broker the sidecar talks to is authenticated: the chart generates a random `N8N_RUNNERS_AUTH_TOKEN` on first install, stores it in a Secret (`n8n-task-runners`), and reuses it across upgrades via Helm's `lookup` function, so rolling the release doesn't invalidate runners mid-flight.

The runner definitions for both languages are already baked into `/etc/n8n-task-runners.json` inside the image.

### Binary Data in Database Mode

n8n supports several binary data storage backends. The chart's built-in storage configuration only covers `filesystem` and `s3` — but `filesystem` is incompatible with queue mode (workers and webhook processors on different pods can't share a local disk). The correct mode for this setup is `database`, which stores binary data as BLOBs in PostgreSQL.

Since there's no structured field for it, it's set as a raw environment variable via `config.extraEnv`, which is applied identically to the main, worker, and webhook-processor deployments:

```yaml
config:
  extraEnv:
    - name: N8N_DEFAULT_BINARY_DATA_MODE
      value: "database"
```

---

## TLS and Routing

The TLS chain is:

1. **mkcert** generates a local CA and installs it into the system and browser trust stores. The CA certificate and key are stored in Secret `cert-manager/ca-cert` by the `cert-manager` presync hook.
2. **cert-manager** uses a `ClusterIssuer` backed by that secret to issue a wildcard certificate for `*.homelab.localhost`, stored in Secret `default/gateway-cert`.
3. **Envoy Gateway** terminates TLS on port 443 using that cert.
4. Two **HTTPRoute** rules split `n8n.homelab.localhost` traffic between n8n's two service-facing deployments: webhook, form, and MCP paths go to the webhook-processor service, everything else goes to the main service.

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
          value: /webhook/
      - path:
          type: PathPrefix
          value: /webhook-waiting/
      - path:
          type: PathPrefix
          value: /form/
      - path:
          type: PathPrefix
          value: /form-waiting/
      - path:
          type: PathPrefix
          value: /mcp/
    backendRefs:
      - name: n8n-webhook-processor
        port: 5678
  - timeouts:
      request: 0s
    matches:
      - path:
          type: PathPrefix
          value: /
    backendRefs:
      - name: n8n-main
        port: 5678
```

The path split mirrors the routing the chart itself would set up with a plain Kubernetes `Ingress`, reimplemented as Gateway API rules since Envoy Gateway consumes `HTTPRoute` instead. Without the `0s` timeout override, Envoy closes the WebSocket after the default idle timeout and the UI starts showing connection errors. This replaces the Envoy-specific `BackendTrafficPolicy` CRD that was previously required for the same effect.

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

Five secrets are used at runtime:

- **`ca-cert`** (`cert-manager`): mkcert CA certificate and key. cert-manager reads this to sign the wildcard TLS certificate.
- **`valkey-auth`**: contains `redis-password`. Used by both the Valkey chart (ACL auth) and the n8n chart (`redis.passwordSecret`).
- **`n8n-core-secrets`**: contains `N8N_ENCRYPTION_KEY`, `N8N_HOST`, `N8N_PORT`, and `N8N_PROTOCOL`. The chart hardcodes lookups for all four keys on this one secret (`secretRefs.existingSecret`), so all four have to exist even though only the encryption key is actually sensitive. Losing `N8N_ENCRYPTION_KEY` makes all saved credentials unrecoverable — treat it like a master password.
- **`n8n-cnpg-cluster-app`**: auto-created by the CNPG operator. Contains the `password` key for the n8n database user, consumed directly by `database.passwordSecret` — no workaround needed.
- **`n8n-task-runners`**: generated by the chart itself, not by a helmfile hook. Holds the random `N8N_RUNNERS_AUTH_TOKEN` the task-runner sidecars use to authenticate to the broker in the main/worker containers.

---

## Security Hardening

Several security measures are applied across all pod types, mostly as raw environment variables under `config.extraEnv` since the chart doesn't expose them as dedicated fields:

| Measure | Value |
| --- | --- |
| Public REST API | Disabled (`N8N_PUBLIC_API_DISABLED: true`) |
| Community packages | Disabled |
| SSRF protection | Enabled — blocks requests to RFC 1918 / loopback addresses from workflow nodes |
| Secure cookie | Enabled — HTTPS-only, SameSite-strict |
| Telemetry | Fully disabled (diagnostics, version notifications, template fetching, frontend hooks) |
| Task runner sidecar | Distroless image, non-root (UID 1000 via the pod's security context), no privilege escalation, all Linux capabilities dropped |

SSRF protection is worth highlighting in a homelab context: without it, a workflow node could make HTTP requests to other services on your local network (your NAS, router admin panel, other Kubernetes services). Enabling this blocks that class of attack even from workflows you write yourself — a useful guardrail.

---

## Monitoring Readiness

The chart ships no `ServiceMonitor` template at all, so metrics collection isn't wired up out of the box. To activate it, deploy `kube-prometheus-stack` into the `monitoring` namespace and add a hand-written `ServiceMonitor` via a helmfile postsync hook, the same way `HTTPRoute` is applied for n8n today.

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
config:
  extraEnv:
    - name: N8N_SSRF_ALLOWED_HOSTNAMES
      value: gemma-e2b-service.llmkube-system.svc.cluster.local
```

The hostname allowlist takes precedence over the IP blocklist, so n8n skips the RFC 1918 check entirely for this host. The Kubernetes internal DNS zone is fully under our control, which is the intended use case for this allowlist type.

With that in place, any n8n node that accepts an OpenAI credential — AI Agent, Chat, Summarize, etc. — works against the local model. No traffic leaves the machine.

---

## Summary

This setup treats a local homelab deployment with the same seriousness as a production one. Queue mode with dedicated worker and webhook-processor pods, encrypted secrets, SSRF protection, a proper Envoy-based gateway, isolated and authenticated task runners in distroless containers — none of it is strictly necessary for a single-user local install, but all of it is good practice and costs almost nothing given a declarative approach.

The LLMKube layer adds a local inference service that slots into n8n's existing OpenAI credential system. Workflows that use AI nodes run entirely on-cluster: no API keys, no external calls, no data leaving the machine. The pgvector extension in the PostgreSQL cluster is already there if you want to add a vector store to the mix.
