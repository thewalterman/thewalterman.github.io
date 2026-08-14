---
title: "Three OpenTofu Roots for OCI, Tailscale, and Cloudflare"
date: 2026-08-11
draft: false
tags: ["opentofu", "terraform", "oci", "oracle-cloud", "tailscale", "cloudflare", "k3s", "iac", "ansible"]
description: "How the cloud footprint under my homelab is provisioned today — three independent OpenTofu roots for OCI compute/network/IAM, the Tailscale tailnet, and Cloudflare DNS/WAF/tunnel ingress, plus the Ansible layer that still sits outside all three."
ShowToc: true
---

My [homelab post](/posts/homelab/) starts from a working k3s cluster and builds a GitOps stack on top of it. It never explains where the cluster, the tailnet it's reachable over, or the DNS/tunnel it's exposed through actually come from. This post is that layer: a repo of three independent OpenTofu root modules — `oci/`, `tailscale/`, `cloudflare/` — plus the Ansible playbooks that patch the OS and bootstrap k3s itself.

Three roots means three state files, three `tofu init`s, three blast radii. Provisioning OCI compute has nothing to do with editing a Cloudflare WAF rule, and a mistake in one shouldn't require touching the other's state to fix.

---

## `oci/`: Compute, Network, and IAM for the Node(s)

The compute node is an Ampere `VM.Standard.A1.Flex` — Oracle's Always Free ARM shape — defined once and expanded via `for_each` over a map of node definitions:

```hcl
variable "k3s_nodes" {
  type = map(object({
    role          = string
    fault_domain  = optional(string)
    private_ip    = optional(string)
    ocpus         = optional(number, 2)
    memory_in_gbs = optional(number, 12)
  }))
}

resource "oci_core_instance" "node" {
  for_each = var.k3s_nodes

  fault_domain = coalesce(each.value.fault_domain, var.fault_domain)
  display_name = each.key
  shape        = "VM.Standard.A1.Flex"

  shape_config {
    memory_in_gbs = each.value.memory_in_gbs
    ocpus         = each.value.ocpus
    vcpus         = each.value.ocpus # A1.Flex is always 1:1 vcpu:ocpu
  }
  # ...
}
```

The map key doubles as `display_name` and `hostname_label`, today's single entry is the same instance originally clicked into existence through the OCI Console, remapped via a `moved` block so it was never destroyed and recreated when `for_each` was introduced.

Scaling to HA or adding workers is adding entries to `k3s_nodes` with `role = "agent"` and a `fault_domain` spread across `FAULT-DOMAIN-1/2/3` — this region has one availability domain, so fault domains are the only redundancy axis available. Oracle halved the Always Free Ampere allowance; the single node here is already sized at 2 OCPU / 12GB, so it consumes the *entire* free allowance on its own — any additional node means paying for it.

Networking follows the same shared-resource shape as before: one VCN, one subnet, and one default security list carrying every ingress rule the k3s node(s) need — api-server (6443), kubelet (10250), flannel VXLAN (8472/udp), node-exporter (9100), etcd peer/client (2379-2380), each scoped to the VCN's own CIDR. A new node joining `k3s_nodes` is covered by every rule automatically, no per-instance NSG wiring.

What's no longer there: **the default security list has no ingress rule for 22 or 443 anymore.** Both were removed once Tailscale and the Cloudflare Tunnel were confirmed reachable without them — the full debugging story is its own post: [Locking Down the Homelab](/posts/homelab-lockdown/).

IAM hasn't changed in shape: a dynamic group whose matching rule is generated from the same `for_each` map as the instances, so it always covers exactly the nodes that exist —

```hcl
resource "oci_identity_dynamic_group" "dynamic_group" {
  matching_rule = "ANY {${join(", ", [
    for k, n in oci_core_instance.node : "instance.id = '${n.id}'"
  ])}}"
}
```

— paired with a policy granting it read access to secrets and vaults tenancy-wide, and a second policy scoping write access to exactly the backup bucket. This dynamic group is the mechanism the [homelab post](/posts/homelab/) glosses over as "authenticating with the compute instance's own cloud identity" — the `ClusterSecretStore` on k3s reads through this exact policy, no static credential involved.

A small KMS vault and AES-256 key back the secrets the node reads through that dynamic group; `secrets.tf` declares them as `oci_vault_secret`, `for_each` over a `vault_secrets` map, most of them imported from secrets that already existed in the OCI Console before this resource did. The one non-obvious trap: OCI's API never returns secret content back, so without mitigation `tofu plan` would show every one of them as an in-place update — a new version, not a real change — on every single run, forever:

```hcl
resource "oci_vault_secret" "secret" {
  for_each = local.vault_secret_names
  # ...
  secret_content {
    content_type = "BASE64"
    content      = base64encode(var.vault_secrets[each.value].content)
  }

  lifecycle {
    ignore_changes = [secret_content]
  }
}
```

`ignore_changes = [secret_content]` suppresses that — the tradeoff being that changing a value in `terraform.tfvars` after initial creation does nothing until that line is removed or the resource is tainted. Worth knowing before you edit a secret and wonder why `tofu plan` shows nothing.

---

## Provisioning New Nodes Onto the Tailnet at Boot

The one new piece in `oci/`: a `tailscale` provider living alongside `oci` in the same root, whose only job is minting a join key for future nodes.

```hcl
resource "tailscale_tailnet_key" "k3s_node" {
  count = var.tailscale_oauth_client_id != null ? 1 : 0

  reusable            = true
  ephemeral           = false
  preauthorized       = true
  expiry              = 7776000 # 90 days
  tags                = ["tag:k3s-node"]
  recreate_if_invalid = "always"
}
```

`count` keyed off an optional variable means `oci/` works with zero Tailscale credentials at all if you never set `tailscale_oauth_client_id`/`_secret` — new nodes then need a manual Tailscale setup. Set it, and every *new* `k3s_nodes` entry gets a cloud-init `user_data` that installs Tailscale and joins the tailnet unattended at first boot:

```hcl
locals {
  k3s_node_user_data = {
    for name, node in var.k3s_nodes : name => (
      name == "instance-name" || length(tailscale_tailnet_key.k3s_node) == 0
      ? null
      : base64encode(<<-EOT
        #!/bin/bash
        curl -fsSL https://tailscale.com/install.sh | sh
        tailscale up --authkey=${tailscale_tailnet_key.k3s_node[0].key} --ssh --hostname=${name}
      EOT
      )
    )
  }
}
```

The pre-existing node is excluded by name, deliberately: cloud-init only runs on first boot, so setting `user_data` on an already-running instance would be a permanent no-op diff, not a real effect. The join key requires `tag:k3s-node` to already exist in the tailnet's ACL (see below) and an OAuth client scoped narrowly to `auth_keys` and tagged the same way — kept separate from whatever credential manages the rest of the tailnet, on the principle that a key that can only mint join tokens for one tag is a much smaller thing to leak than one that can rewrite the whole ACL.

---

## `tailscale/`: The Tailnet Config That Used to Live Only in a Browser Tab

Before this root existed, the tailnet's ACL, DNS settings, and device key policy were whatever they happened to be clicked to in the Tailscale admin console. `tailscale/` codifies all of it: `tailscale_acl` for the policy file, `tailscale_dns_preferences`/`tailscale_dns_search_paths` for Magic DNS, `tailscale_tailnet_settings` for tailnet-wide toggles (`devices_key_duration_days = 180`, `users_approval_on = true`, and so on), and `tailscale_device_key` entries pinning per-device key expiry by hardcoded `device_id`.

Those `device_id`s aren't copy-pasted from the console — a `data "tailscale_devices" "all"` plus a small output does the lookup:

```hcl
data "tailscale_devices" "all" {}

output "devices" {
  value = { for d in data.tailscale_devices.all.devices : d.name => { id = d.node_id, tags = d.tags } }
}
```

`tofu output devices` turns a device's name into the ID the next `tailscale_device_key` block needs. The k3s node's own key sets `key_expiry_disabled = true` — that device never has to re-authenticate — which is a different mechanism from the reusable `tailscale_tailnet_key` `oci/` mints for onboarding *new* nodes: one pins an already-joined device's key forever, the other hands out fresh join tokens.

---

## `cloudflare/`: DNS, WAF, and Tunnel Ingress, Migrated In

It manages DNS records, a WAF custom firewall ruleset, and the ingress config for a Cloudflare Tunnel, all via the `cloudflare/cloudflare` provider.

The tunnel config looks up an existing Cloudflare Tunnel by name (the tunnel itself is provisioned outside Tofu via `cloudflared`) and manages its ingress rules and the wildcard DNS record pointing at it, sharing one data source between `dns.tf` and `tunnel.tf`:

```hcl
data "cloudflare_zero_trust_tunnel_cloudflared" "homelab" {
  account_id = var.cloudflare_account_id
  filter = { name = "homelab" }
}

resource "cloudflare_zero_trust_tunnel_cloudflared_config" "homelab" {
  account_id = var.cloudflare_account_id
  tunnel_id  = data.cloudflare_zero_trust_tunnel_cloudflared.homelab.id

  config = {
    ingress = [
      {
        hostname = "*.example.com"
        service  = "https://envoy-gateway-proxy.envoy-gateway-system.svc.cluster.local:443"
        origin_request = {
          origin_server_name = "tunnel.example.com"
        }
      },
      { service = "http_status:404" }
    ]
  }
}
```

That `origin_server_name` isn't decorative — without it, the TLS handshake against Envoy Gateway's SNI-based routing fails outright. `mise.toml` additionally pins `cf-terraforming`, used to regenerate resource declarations from live dashboard drift when something gets clicked instead of committed.

---

## Ansible Still Sits Outside All Three

k3s itself is installed and upgraded entirely outside any of these three roots — no `oci_core_instance` provisioning block, no cloud-init install step for the pre-existing node. Two playbooks, one inventory:

```bash
# OS-level maintenance
ansible-playbook -i ansible/inventory.yml ansible/update.yml

# (Re-)install k3s via the external k3s-ansible collection
ansible-galaxy collection install -r ansible/requirements.yml
ansible-playbook -i ansible/inventory.yml ansible/bootstrap-k3s.yml
```

Both playbooks now proxy SSH through the Tailscale sidecar rather than reaching the node's public IP directly on 22, since that ingress rule no longer exists (`ansible_ssh_common_args` with a `docker exec -i tailscale nc %h %p` `ProxyCommand`) — the sidecar has to be up and authenticated before either runs. The sharp edges inside `bootstrap-k3s.yml` are unchanged: `token` must match what's already on the node or a real run silently rewrites the cluster's join token, and `server_config_yaml` must mirror `/etc/rancher/k3s/config.yaml` byte-for-byte in intent, since a real run always overwrites that file from the variable rather than merging it.

---

## What's Provisioned

| Layer | Tool | What |
| --- | --- | --- |
| Compute | OpenTofu (`oci/`) | ARM `A1.Flex` instance(s), `for_each` over `k3s_nodes` |
| Network | OpenTofu (`oci/`) | VCN, subnet, shared default security list — no 22 or 443 ingress |
| IAM & secrets | OpenTofu (`oci/`) | Self-building dynamic group, KMS vault/key, `oci_vault_secret` per secret |
| Backups | OpenTofu (`oci/`) | Object Storage bucket |
| Node tailnet join | OpenTofu (`oci/`) | Optional `tailscale_tailnet_key`, cloud-init `user_data` for new nodes |
| Tailnet config | OpenTofu (`tailscale/`) | ACL/policy file, DNS, tailnet settings, device key expiry |
| DNS, WAF, tunnel | OpenTofu (`cloudflare/`) | DNS records, IP-allowlist WAF ruleset, Tunnel ingress config |
| OS maintenance | Ansible | apt upgrade/autoclean/autoremove, conditional reboot |
| k3s lifecycle | Ansible | k3s-ansible collection, version-pinned install/upgrade |

---

## Conclusion

None of the individual pieces are exciting — a VM, a tailnet policy, a WAF allowlist. What's worth writing up is that each has its own state and its own narrow job.

OpenTofu's job in `oci/` stops the moment the instance and its identity exist; `tailscale/`'s stops at the tailnet's own configuration; `cloudflare/`'s stops at DNS and the edge.

Ansible picks up from there to get k3s running, and everything in the [homelab post](/posts/homelab/) — Flux, Keycloak — starts once that handoff is done. Four narrow responsibilities, four clean boundaries.
