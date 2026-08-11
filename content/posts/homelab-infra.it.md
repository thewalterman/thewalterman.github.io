---
title: "Tre Root OpenTofu per OCI, Tailscale e Cloudflare"
date: 2026-08-11
draft: false
tags: ["opentofu", "terraform", "oci", "oracle-cloud", "tailscale", "cloudflare", "k3s", "iac", "ansible"]
description: "Come viene fatto oggi il provisioning dell'infrastruttura cloud sotto il mio homelab — tre root module OpenTofu indipendenti per compute/network/IAM su OCI, il tailnet Tailscale, e DNS/WAF/tunnel su Cloudflare, più il livello Ansible che resta fuori da tutti e tre."
ShowToc: true
---

Il mio [post sull'homelab](/posts/homelab/) parte da un cluster k3s già funzionante e ci costruisce sopra uno stack GitOps. Non spiega mai da dove arrivino il cluster, il tailnet attraverso cui è raggiungibile, o il DNS/tunnel attraverso cui è esposto. Questo post è quello strato: un repository di tre root module OpenTofu indipendenti — `oci/`, `tailscale/`, `cloudflare/` — più i playbook Ansible che aggiornano il sistema operativo e fanno il bootstrap di k3s stesso.

Tre root significano tre state file, tre `tofu init`, tre raggi d'azione separati. Fare il provisioning del compute su OCI non ha nulla a che fare con la modifica di una regola WAF su Cloudflare, e un errore in uno non dovrebbe richiedere di toccare lo state dell'altro per essere corretto.

---

## `oci/`: Compute, Network e IAM per il/i Nodo/i

Il nodo compute è un Ampere `VM.Standard.A1.Flex` — la shape ARM Always Free di Oracle — definito una sola volta ed espanso tramite `for_each` su una mappa di definizioni di nodo:

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
    vcpus         = each.value.ocpus # A1.Flex è sempre 1:1 vcpu:ocpu
  }
  # ...
}
```

La chiave della mappa funge anche da `display_name` e `hostname_label`, l'unica voce di oggi è la stessa istanza creata originariamente a colpi di click nella Console OCI, rimappata tramite un blocco `moved` così da non essere mai stata distrutta e ricreata quando è stato introdotto `for_each`.

Scalare verso l'HA o aggiungere worker significa aggiungere voci a `k3s_nodes` con `role = "agent"` e un `fault_domain` distribuito su `FAULT-DOMAIN-1/2/3` — questa regione ha un solo availability domain, quindi i fault domain sono l'unico asse di ridondanza disponibile. Oracle ha dimezzato l'allowance Always Free su Ampere per l'intera tenancy; l'unico nodo qui è già dimensionato a 2 OCPU / 12GB, quindi consuma da solo *l'intera* allowance gratuita — qualsiasi nodo aggiuntivo va pagato.

Il networking segue la stessa forma a risorse condivise di prima: una VCN, una subnet, e un'unica security list di default che porta ogni regola di ingress di cui il/i nodo/i k3s hanno bisogno — api-server (6443), kubelet (10250), il VXLAN di flannel (8472/udp), node-exporter (9100), etcd peer/client (2379-2380), ciascuna limitata al CIDR della VCN stessa. Un nuovo nodo aggiunto a `k3s_nodes` è coperto automaticamente da ogni regola, senza collegamenti NSG per istanza.

Quello che non c'è più: **la security list di default non ha più nessuna regola di ingress per 22 o 443.** Entrambe sono state rimosse una volta confermato che Tailscale e il Cloudflare Tunnel fossero raggiungibili senza — la storia completa del debugging è un post a parte: [Blindare l'Homelab](/posts/homelab-lockdown/).

L'IAM non è cambiata nella forma: un dynamic group la cui matching rule viene generata dalla stessa mappa `for_each` delle istanze, quindi copre sempre esattamente i nodi che esistono —

```hcl
resource "oci_identity_dynamic_group" "dynamic_group" {
  matching_rule = "ANY {${join(", ", [
    for k, n in oci_core_instance.node : "instance.id = '${n.id}'"
  ])}}"
}
```

— abbinata a una policy che concede accesso in lettura a segreti e vault per l'intera tenancy, e a una seconda policy che limita l'accesso in scrittura esattamente al bucket di backup. Questo dynamic group è il meccanismo che il [post sull'homelab](/posts/homelab/) liquida con "autenticandosi con l'identità cloud propria dell'istanza compute" — il `ClusterSecretStore` su k3s legge proprio attraverso questa policy, senza nessuna credenziale statica coinvolta.

Un piccolo vault KMS e una chiave AES-256 sono ciò che sta dietro i segreti che il nodo legge tramite quel dynamic group; `secrets.tf` li dichiara come `oci_vault_secret`, `for_each` su una mappa `vault_secrets`, la maggior parte importati da segreti che esistevano già nella Console OCI prima che questa risorsa esistesse. L'unica trappola non ovvia: l'API di OCI non restituisce mai indietro il contenuto di un secret, quindi senza mitigazione `tofu plan` mostrerebbe ognuno di essi come un aggiornamento in-place — una nuova versione, non una modifica reale — ad ogni singola esecuzione, per sempre:

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

`ignore_changes = [secret_content]` sopprime questo comportamento — il compromesso è che modificare un valore in `terraform.tfvars` dopo la creazione iniziale non fa nulla finché quella riga non viene rimossa o la risorsa non viene "taintata". Meglio saperlo prima di modificare un secret e chiedersi perché `tofu plan` non mostri nulla.

---

## Far Unire il Tailnet ai Nuovi Nodi al Boot

L'unico pezzo nuovo in `oci/`: un provider `tailscale` che vive accanto a `oci` nello stesso root, il cui unico compito è generare una join key per i nodi futuri.

```hcl
resource "tailscale_tailnet_key" "k3s_node" {
  count = var.tailscale_oauth_client_id != null ? 1 : 0

  reusable            = true
  ephemeral           = false
  preauthorized       = true
  expiry              = 7776000 # 90 giorni
  tags                = ["tag:k3s-node"]
  recreate_if_invalid = "always"
}
```

Un `count` legato a una variabile opzionale significa che `oci/` funziona con zero credenziali Tailscale se non si impostano `tailscale_oauth_client_id`/`_secret` — i nuovi nodi allora richiedono una configurazione manuale di Tailscale. Impostandola, ogni *nuova* voce in `k3s_nodes` riceve uno `user_data` di cloud-init che installa Tailscale e unisce il nodo al tailnet non presidiato al primo boot:

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

Il nodo pre-esistente è escluso per nome, deliberatamente: cloud-init gira solo al primo boot, quindi impostare `user_data` su un'istanza già in esecuzione sarebbe un no-op permanente nel diff, non un effetto reale. La join key richiede che `tag:k3s-node` esista già nell'ACL del tailnet (vedi sotto) e un OAuth client con lo scope limitato ad `auth_keys` e taggato allo stesso modo — tenuto separato da qualunque credenziale gestisca il resto del tailnet, sul principio che una chiave capace solo di generare join token per un tag è qualcosa di molto più piccolo da perdere rispetto a una capace di riscrivere l'intera ACL.

---

## `tailscale/`: La Configurazione del Tailnet Che Prima Viveva Solo in una Scheda del Browser

Prima che questo root esistesse, l'ACL del tailnet, le impostazioni DNS e la policy sulle chiavi dei device erano qualunque cosa fosse stata impostata a colpi di click nella console admin di Tailscale. `tailscale/` codifica tutto questo: `tailscale_acl` per il policy file, `tailscale_dns_preferences`/`tailscale_dns_search_paths` per Magic DNS, `tailscale_tailnet_settings` per le impostazioni a livello di tailnet (`devices_key_duration_days = 180`, `users_approval_on = true`, e così via), e voci `tailscale_device_key` che fissano la scadenza della chiave per singolo device tramite `device_id` hardcoded.

Quei `device_id` non vengono copiati a mano dalla console — un `data "tailscale_devices" "all"` più un piccolo output fanno il lookup:

```hcl
data "tailscale_devices" "all" {}

output "devices" {
  value = { for d in data.tailscale_devices.all.devices : d.name => { id = d.node_id, tags = d.tags } }
}
```

`tofu output devices` trasforma il nome di un device nell'ID che serve al prossimo blocco `tailscale_device_key`. La chiave del nodo k3s ha `key_expiry_disabled = true` — quel device non deve mai ri-autenticarsi — un meccanismo diverso dalla `tailscale_tailnet_key` riusabile che `oci/` genera per l'onboarding di *nuovi* nodi: una fissa per sempre la chiave di un device già unito, l'altra distribuisce join token nuovi.

---

## `cloudflare/`: DNS, WAF e Ingress del Tunnel, Migrati Dentro

Gestisce record DNS, un ruleset WAF custom, e la configurazione di ingress per un Cloudflare Tunnel, tutto tramite il provider `cloudflare/cloudflare`.

La configurazione del tunnel cerca un Cloudflare Tunnel esistente per nome (il tunnel stesso è provisionato al di fuori di Tofu tramite `cloudflared`) e gestisce le sue regole di ingress e il record DNS wildcard che vi punta, condividendo un'unica data source tra `dns.tf` e `tunnel.tf`:

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
        service  = "https://envoy-gateway.envoy-gateway-system.svc.cluster.local:443"
        origin_request = {
          origin_server_name = "tunnel.example.com"
        }
      },
      { service = "http_status:404" }
    ]
  }
}
```

Quell'`origin_server_name` non è decorativo — senza, l'handshake TLS contro il routing basato su SNI di Envoy Gateway fallisce del tutto. `mise.toml` fissa anche `cf-terraforming`, usato per rigenerare le dichiarazioni delle risorse a partire da drift live sulla dashboard, quando qualcosa viene cliccato invece che committato.

---

## Ansible Resta Fuori da Tutti e Tre

k3s in sé viene installato e aggiornato interamente al di fuori di questi tre root — nessun blocco di provisioning `oci_core_instance`, nessuno step di installazione via cloud-init per il nodo pre-esistente. Due playbook, un unico inventory:

```bash
# Manutenzione a livello OS
ansible-playbook -i inventory.yml update.yml

# (Re-)installa k3s tramite la collection esterna k3s-ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory.yml bootstrap-k3s.yml
```

Entrambi i playbook ora instradano SSH attraverso il sidecar Tailscale invece di raggiungere direttamente l'IP pubblico del nodo sulla 22, dato che quella regola di ingress non esiste più (`ansible_ssh_common_args` con un `ProxyCommand` `docker exec -i tailscale nc %h %p`) — il sidecar deve essere attivo e autenticato prima di eseguire uno dei due. Gli spigoli vivi dentro `bootstrap-k3s.yml` restano invariati: `token` deve corrispondere a quello già presente sul nodo o un run reale riscrive silenziosamente il join token del cluster, e `server_config_yaml` deve rispecchiare `/etc/rancher/k3s/config.yaml` byte per byte nell'intento, perché un run reale sovrascrive sempre quel file a partire dalla variabile invece di unirlo.

---

## Di Cosa Viene Fatto il Provisioning

| Livello | Strumento | Cosa |
| --- | --- | --- |
| Compute | OpenTofu (`oci/`) | Istanza/e ARM `A1.Flex`, `for_each` su `k3s_nodes` |
| Rete | OpenTofu (`oci/`) | VCN, subnet, security list di default condivisa — nessun ingress su 22 o 443 |
| IAM e segreti | OpenTofu (`oci/`) | Dynamic group auto-costruito, vault/chiave KMS, `oci_vault_secret` per segreto |
| Backup | OpenTofu (`oci/`) | Bucket Object Storage |
| Ingresso al tailnet | OpenTofu (`oci/`) | `tailscale_tailnet_key` opzionale, `user_data` cloud-init per nuovi nodi |
| Configurazione tailnet | OpenTofu (`tailscale/`) | ACL/policy file, DNS, impostazioni tailnet, scadenza chiavi device |
| DNS, WAF, tunnel | OpenTofu (`cloudflare/`) | Record DNS, ruleset WAF allowlist, config ingress del Tunnel |
| Manutenzione OS | Ansible | apt upgrade/autoclean/autoremove, reboot condizionale |
| Ciclo di vita k3s | Ansible | Collection k3s-ansible, install/upgrade con versione pinnata |

---

## Conclusione

Nessuno dei singoli pezzi è entusiasmante — una VM, una policy di tailnet, un allowlist WAF. Ciò che vale la pena raccontare è che ciascuno ha il proprio state e il proprio compito ristretto.

Il compito di OpenTofu in `oci/` finisce nel momento in cui l'istanza e la sua identità esistono; quello di `tailscale/` finisce alla configurazione del tailnet stesso; quello di `cloudflare/` finisce a DNS ed edge. 

Ansible prende in carico da lì per portare k3s in esecuzione, e tutto ciò che è descritto nel [post sull'homelab](/posts/homelab/) — Flux, Keycloak — parte una volta completato quel passaggio di consegne. Quattro responsabilità ristrette, quattro confini puliti.
