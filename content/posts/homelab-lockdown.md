---
title: "Locking Down the Homelab: Closing All Ports"
date: 2026-08-10
draft: false
tags: ["cloudflare", "tailscale", "k3s", "docker", "security", "networking", "opentofu", "homelab"]
description: "Closing the public ports left open on the homelab node — 443 behind a Cloudflare Tunnel with an SNI mismatch bug to chase, and 22 behind a Tailscale sidecar in Docker with a userspace-networking trap along the way."
ShowToc: true
---

The [homelab](/posts/homelab/) and its [OCI infrastructure](/posts/homelab-infra/) have been running with two ports open to the internet: 443, proxied through Cloudflare but still reachable directly on the node behind it, and 22, plain SSH on the public IP. Both were still attack surface that didn't need to exist. This is the writeup of closing both — one port at a time, with a debugging detour on each.

---

## Part 1: Cloudflare Tunnel — Closing 443

### Two tunnel charts, opposite philosophies

Cloudflare's own [`cloudflare/helm-charts`](https://github.com/cloudflare/helm-charts) repo ships two charts for the same job, built on opposite models:

| | `cloudflare-tunnel` (locally-managed) | `cloudflare-tunnel-remote` |
| --- | --- | --- |
| Routing config | `values.yaml` — ConfigMap in-cluster, ingress rules in git | Cloudflare dashboard/API, outside the cluster |
| k8s resources | Deployment + Secret + ConfigMap | Deployment + Secret (token only) |
| Credentials needed | account, tunnel name, tunnel ID, secret | a single tunnel token |

**Decision: `cloudflare-tunnel-remote`.** Under 50 lines of templates total, the `cloudflared` image floats on `latest` by default so it never goes stale, and it keeps almost no Cloudflare-specific config versioned in the repo. The tradeoff: the ingress rules live in Cloudflare's dashboard/API instead of git. For one wildcard route into the cluster Gateway, that was worth it.

**A side effect worth flagging**: cloudflared needs a stable name to reach Envoy Gateway's Service inside the cluster, but Envoy Gateway generates a non-deterministic, hash-suffixed Service name by default. Fixed with an `EnvoyProxy` resource, pinning the Service name to `envoy-gateway`.

### Bad Gateway: an SNI mismatch

Routing `*.example.com` to `https://envoy-gateway.envoy-gateway-system.svc.cluster.local:443` produced a flat **502 Bad Gateway** on every hostname:

```
error="Unable to reach the origin service. [...]: read tcp [...]: connection reset by peer"
originService=http://envoy-gateway.envoy-gateway-system.svc.cluster.local:443
```

The cause: Envoy Gateway picks its TLS filter chain by matching the client's SNI, and only names under `*.example.com` match anything. Without an explicit `origin_server_name`, cloudflared's TLS client sends the *internal Service hostname* as SNI (`envoy-gateway.envoy-gateway-system.svc.cluster.local`) — that matches no filter chain, so Envoy resets the connection before the handshake even completes.

The routing to the right backend inside Envoy depends on the request's `Host` header, not on the SNI used for the handshake — so any static hostname under `*.example.com` works as `origin_server_name`, whether or not it has a real DNS record or route (`tunnel.example.com` here). It just needs to survive the TLS handshake; per-app routing keeps working off `Host`.

### Applying the fix with OpenTofu

The tunnel's ingress config, including `origin_server_name`, is managed entirely through OpenTofu — the same tool already managing the rest of this Cloudflare zone (DNS, WAF), with `cloudflare_zero_trust_tunnel_cloudflared_config` as the resource for it.

```hcl
data "cloudflare_zero_trust_tunnel_cloudflared" "homelab" {
  account_id = var.cloudflare_account_id
  filter = {
    name = "homelab"
  }
}

resource "cloudflare_zero_trust_tunnel_cloudflared_config" "homelab" {
  account_id = var.cloudflare_account_id
  tunnel_id  = data.cloudflare_zero_trust_tunnel_cloudflared.homelab.id

  config = {
    ingress = [
      {
        hostname = "*.example.com"
        service  = "https://envoy-gateway.envoy-gateway-system.svc.cluster.local:443"
        origin_request = {
          origin_server_name = "tunnel.example.com"
        }
      },
      {
        service = "http_status:404"
      }
    ]
  }
}
```

The trailing rule with no `hostname` is a required fallback — the provider rejects the config without it. After `tofu apply`, `originServerName` showed up in the cloudflared logs, and `curl` against `monitoring.example.com` stopped resetting and started getting a real response.

The wildcard's old **A** record — a leftover from before the tunnel, pointing straight at the public IP — was swapped for a **CNAME** pointed at the tunnel instead:

```hcl
resource "cloudflare_dns_record" "example_com_wildcard" {
  zone_id  = var.zone_id
  name     = "*.example.com"
  type     = "CNAME"
  content  = "${data.cloudflare_zero_trust_tunnel_cloudflared.homelab.id}.cfargotunnel.com"
  ttl      = 1
  proxied  = true
  tags     = []
  settings = {}
}
```

Final `tofu plan`: clean.

With traffic confirmed flowing through the tunnel, 443 was removed from the default security list.

---

## Part 2: SSH via Tailscale — Closing 22

The other open port was 22. Tailscale was already installed on the node, via the official install script, and joined to the tailnet as a mesh VPN. The goal was to reach it over Tailscale via Docker then close 22 to the internet entirely.

### A Tailscale sidecar in Docker

Standard pattern from [Tailscale's own docs](https://tailscale.com/docs/features/containers/docker/how-to/connect-docker-container): a container running the `tailscale/tailscale` image joins the tailnet and acts as a network sidecar for other containers.

```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale
    hostname: docker-ssh-client
    environment:
      - TS_AUTHKEY=<tskey-...>
      - TS_STATE_DIR=/var/lib/tailscale
    volumes:
      - ./tailscale-state:/var/lib/tailscale
    cap_add:
      - net_admin
      - net_raw
    restart: unless-stopped
```

### The bug: userspace networking

The TCP connection just timed out — no "connection refused," no password prompt, total silence. Diagnosis:

```bash
docker exec tailscale ps aux | grep tailscaled
# tailscaled --tun=userspace-networking   <-- here
```

Without a TUN device passed into the container, `tailscaled` defaults to **userspace networking**: no real `tailscale0` interface in the network namespace. Tailscale's own commands (`tailscale ping`) still work — they go through `tailscaled`'s internal stack — which is exactly what makes this trap insidious: the tool that would normally prove connectivity keeps working right through the actual problem. Only `ps aux` and `ip a` inside the sidecar reveal it. Ordinary kernel sockets — `ssh`, `nc`, anything from a container sharing the network — have nowhere to go, hence the silent timeout.

Fix, and both parts are required together:

```yaml
services:
  tailscale:
    # ...
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - TS_USERSPACE=false   # without this, the tun device is passed in but ignored
```

With Tailscale access confirmed, the public SSH ingress rule was removed from the default security list.

---

## What's actually closed now

| Port | Was | Now |
| --- | --- | --- |
| 443 | Open on the node, Cloudflare proxying in front | Closed — traffic reaches the node only via the Cloudflare Tunnel |
| 22 | Open to `0.0.0.0/0` | Closed — access only via Tailscale, decapsulated locally over already-allowed UDP |

## Takeaways

- **Cloudflare Tunnel**: the "remote" chart wins for a homelab specifically *because* it keeps almost nothing Cloudflare-specific in git — the tradeoff is that the ingress rules live in Cloudflare's API rather than a ConfigMap, which is exactly why managing them through OpenTofu from day one matters.
- **SNI vs `Host`**: Envoy's filter-chain selection runs on the TLS SNI, while its actual routing runs on the HTTP `Host` header — two different fields doing two different jobs, easy to conflate when only one of them is failing.
- **Tailscale in Docker**: userspace networking is the trap that hides behind Tailscale's own diagnostics still working while everything else times out silently.
- Both fixes share the same shape: encapsulated/tunneled traffic (Cloudflare Tunnel's outbound connection, Tailscale's UDP) never touches the security list or firewall rule being deleted — which is exactly why closing the port is safe in the first place, not an assumption to leave unverified.
