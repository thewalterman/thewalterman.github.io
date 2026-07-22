---
title: "The Infrastructure Under the Homelab: Provisioning OCI with OpenTofu and Ansible"
date: 2026-07-21
draft: false
tags: ["opentofu", "terraform", "oci", "oracle-cloud", "k3s", "iac", "ansible"]
description: "How the k3s node behind my homelab actually gets created — a single OpenTofu root module for OCI compute, networking, and IAM, plus the Ansible playbooks that patch the OS and bootstrap k3s itself."
ShowToc: true
---

My [homelab post](/posts/homelab/) starts from a working k3s cluster and builds a full GitOps stack on top of it — FluxCD, Keycloak, an external vault reached over instance principal auth. It never explains where that cluster, or the vault, actually come from. This post is the layer underneath: a single OpenTofu root module that provisions the OCI compute, networking, and IAM for k3s, and a pair of Ansible playbooks that patch the OS and drive the k3s install itself.

There are no submodules here. Every `.tf` file lives at the repo root and shares one implicit provider and one state file. For a homelab with one tenancy and one cluster, splitting this into modules would just be indirection — a flat root module is easier to read end to end, and that's worth more than reusability nobody needs yet.

---

## Compute: One `for_each`, One ARM Node

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

  fault_domain   = coalesce(each.value.fault_domain, var.fault_domain)
  display_name   = each.key
  shape          = "VM.Standard.A1.Flex"

  shape_config {
    memory_in_gbs = each.value.memory_in_gbs
    ocpus         = each.value.ocpus
    vcpus         = each.value.ocpus # A1.Flex is always 1:1 vcpu:ocpu
  }
  # ...
}
```

The map key doubles as `display_name` and `hostname_label`, so it has to be a valid hostname — that's why nodes are named `instance-20260402-1401` rather than something readable. Today there's exactly one entry, and it isn't a fresh resource: it's the same instance originally clicked into existence through the OCI Console, remapped from a non-`for_each` `oci_core_instance.instance` address via a `moved` block:

```hcl
moved {
  from = oci_core_instance.instance
  to   = oci_core_instance.node["instance-20260402-1401"]
}
```

That one line is the difference between "refactor to `for_each`" and "destroy and recreate the only node the cluster has." Scaling to HA or adding workers is just adding entries to `k3s_nodes` with `role = "agent"` and a `fault_domain` spread across `FAULT-DOMAIN-1/2/3` — this region only has one availability domain, so fault domains are the only redundancy axis available. Oracle recently halved the Always Free Ampere allowance on 2026-06-15, from 4 OCPU / 24GB to 2 OCPU / 12GB tenancy-wide. The single node here is already sized at 2 OCPU / 12GB, so it now consumes the *entire* free allowance on its own.

---

## Networking: Shared Security List, Not Per-Instance Rules

The VCN, subnet, DHCP options, route table, and default security list are all managed via `manage_default_resource_id` rather than as separate custom resources — the pattern OCI's Console leaves behind when you edit the defaults instead of creating new ones. There are also two NSGs (`nsg_quick_action_1`/`_2`) that are pure console leftovers: egress-only, no ingress rules, artifacts of clicking "Quick Actions" in the UI early on.

All the real ingress rules live on the one default security list, shared by the whole subnet:

```hcl
ingress_security_rules {
  description = "etcd peer/client"
  protocol    = "6"
  source      = "10.0.0.0/16"
  source_type = "CIDR_BLOCK"
  tcp_options {
    max = "2380"
    min = "2379"
  }
}
```

ssh, https, the k3s API server (6443), kubelet (10250), flannel VXLAN (8472/udp), node-exporter (9100), and etcd peer/client (2379-2380) each get their own block, scoped to the VCN CIDR where the port has no business being reached from the internet. Because it's one shared list rather than per-instance security groups, a new node joining `k3s_nodes` is covered by every existing rule automatically — no per-instance NSG wiring.

One non-obvious trap: `ingress_security_rules` is a Terraform/OpenTofu `set`, and its `description` field is optional-but-computed. Leave `description` unset on a *new* rule while every existing rule already sets one, and `tofu plan` shows every other rule as removed-and-re-added — a cosmetic set-diff artifact, not a real change, but alarming the first time you see a plan claim it's touching seven rules you didn't ask about. Setting `description` explicitly on every rule, always, avoids it entirely.

---

## IAM: A Dynamic Group That Builds Itself

The instance needs to read secrets from a vault without embedding a static credential anywhere. OCI does this through **dynamic groups** — a group whose membership is computed from a matching rule against instance metadata, rather than assigned by hand — paired with policies that grant the group specific permissions.

The matching rule is generated from the same `for_each` map as the instances, so it always covers exactly the nodes that exist:

```hcl
resource "oci_identity_dynamic_group" "dynamic_group" {
  matching_rule = "ANY {${join(", ", [
    for k, n in oci_core_instance.node : "instance.id = '${n.id}'"
  ])}}"
}
```

Add a node to `k3s_nodes`, and it's automatically in the dynamic group on the next apply — no separate identity wiring per node. Two policies ride on top of it: one grants read access to secrets and vaults tenancy-wide (`Allow dynamic-group '.../dynamic-group' to read secret-family in tenancy`), the other scopes write access to exactly the backup bucket (`where target.bucket.name='...'`). A third, unrelated policy grants the `Administrators` group tenant-admin — a fixed policy from Console setup, not something derived from the node graph.

This dynamic group is precisely the mechanism the [homelab post](/posts/homelab/) glosses over as "authenticating with the compute instance's own cloud identity rather than any credential stored in the cluster." The `ClusterSecretStore` on k3s using `principalType: InstancePrincipal` is reading through this exact policy.

---

## The Vault and the Backup Bucket

A small KMS vault and a software-protected AES-256 key back the secrets the node reads through that dynamic group:

```hcl
resource "oci_kms_vault" "vault" {
  vault_type = "DEFAULT"
}

resource "oci_kms_key" "key" {
  protection_mode = "SOFTWARE"
  key_shape {
    algorithm = "AES"
    length    = "32"
  }
}
```

Alongside it, one Object Storage bucket (`NoPublicAccess`, no versioning, Standard tier) exists purely as a backup target — the thing CNPG ship snapshots into, and the thing the k3s embedded-etcd datastore can be pointed at directly through its S3-compatible endpoint. That second path is worth calling out: etcd's own snapshot upload doesn't understand instance principals, only static access/secret keys — an OCI "Customer Secret Key" generated under a user, not an API key and not the dynamic group above. It's the one place in this whole setup that still needs a credential sitting in a config file instead of an identity.

---

## Ansible: Patching and Bootstrapping, Deliberately Separate from Tofu

k3s itself isn't part of the OpenTofu state or resource graph — cloud-init never installs it, and there's no `oci_core_instance` provisioning block for it. It's installed and upgraded entirely through Ansible, against an inventory the Tofu outputs help you fill in by hand (`k3s_nodes` output gives role/public_ip/private_ip per node) but don't generate automatically.

Two playbooks, same inventory, two different jobs:

```bash
# OS-level maintenance — apt upgrade, autoremove, autoclean, conditional reboot
ansible-playbook -i inventory.yml update.yml

# (Re-)install k3s via the external k3s-ansible collection
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory.yml bootstrap-k3s.yml
```

`bootstrap-k3s.yml` is a one-line wrapper importing `k3s.orchestration.site` from the [k3s-ansible](https://github.com/k3s-io/k3s-ansible) collection, pinned by git ref in `requirements.yml`. The real configuration lives in `inventory.yml`'s `k3s_cluster` group vars, and two of them are sharp edges:

- **`token`** must match whatever's already on the node (`/var/lib/rancher/k3s/server/token`), or a real run silently rewrites the cluster's join token.
- **`server_config_yaml`** must mirror the node's existing `/etc/rancher/k3s/config.yaml` byte-for-byte in intent, because a real run always overwrites that file from this variable — anything configured on the node but missing from `server_config_yaml` (like `disable: [traefik]`) gets silently dropped, not merged.

`k3s_version` is a deliberate, manual bump — re-running the playbook with a newer pin triggers a real upgrade on a live, single-node cluster, so it's not something to leave drifting.

---

## What's Provisioned

| Layer | Tool | What |
| --- | --- | --- |
| Compute | OpenTofu | ARM `A1.Flex` instance(s), `for_each` over `k3s_nodes` |
| Network | OpenTofu | VCN, subnet, shared default security list, two leftover NSGs |
| IAM | OpenTofu | Self-building dynamic group, secret/vault/backup-bucket policies |
| Secrets | OpenTofu | KMS vault + AES-256 key |
| Backups | OpenTofu | Object Storage bucket (backups) |
| OS maintenance | Ansible | apt upgrade/autoremove/autoclean, conditional reboot |
| k3s lifecycle | Ansible | k3s-ansible collection, version-pinned install/upgrade |

---

## Conclusion

None of this is exciting on its own — a VM, a VCN, a vault, a couple of policies. What makes it worth writing up is the seam: OpenTofu's job stops the moment the instance and its identity exist, Ansible's job stops the moment k3s is running and joined, and everything described in the [homelab post](/posts/homelab/) — Flux, Keycloak, the external vault via instance principal — starts from there. Three tools, three narrow responsibilities, and a clean handoff between them is a better fit for a one-node homelab than one tool trying to own the whole lifecycle.
