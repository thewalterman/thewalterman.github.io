---
title: "L'Infrastruttura Sotto l'Homelab: Provisioning di OCI con OpenTofu e Ansible"
date: 2026-07-21
draft: false
tags: ["opentofu", "terraform", "oci", "oracle-cloud", "k3s", "iac", "ansible"]
description: "Come nasce davvero il nodo k3s dietro il mio homelab — un unico root module OpenTofu per compute, networking e IAM su OCI, più i playbook Ansible che aggiornano il sistema operativo e fanno il bootstrap di k3s."
ShowToc: true
---

Il mio [post sull'homelab](/posts/homelab/) parte da un cluster k3s già funzionante e ci costruisce sopra uno stack GitOps completo — FluxCD, Keycloak, un vault esterno raggiunto tramite instance principal. Non spiega mai da dove arrivino quel cluster, né quel vault. Questo post è lo strato sottostante: un unico root module OpenTofu che fa il provisioning di compute, networking e IAM su OCI per k3s, più due playbook Ansible che aggiornano il sistema operativo e guidano l'installazione di k3s stesso.

Qui non ci sono submodule. Ogni file `.tf` vive nella root del repository e condivide un unico provider implicito e un unico state file. Per un homelab con una sola tenancy e un solo cluster, dividerlo in moduli sarebbe solo indirection — un root module piatto si legge da cima a fondo più facilmente, e questo vale più di una riusabilità di cui nessuno ha ancora bisogno.

---

## Compute: Un Solo `for_each`, Un Solo Nodo ARM

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

  fault_domain   = coalesce(each.value.fault_domain, var.fault_domain)
  display_name   = each.key
  shape          = "VM.Standard.A1.Flex"

  shape_config {
    memory_in_gbs = each.value.memory_in_gbs
    ocpus         = each.value.ocpus
    vcpus         = each.value.ocpus # A1.Flex è sempre 1:1 vcpu:ocpu
  }
  # ...
}
```

La chiave della mappa funge anche da `display_name` e `hostname_label`, quindi deve essere un hostname valido — per questo i nodi si chiamano `instance-20260402-1401` invece di qualcosa di leggibile. Oggi c'è esattamente una voce, e non è una risorsa nuova: è la stessa istanza creata originariamente a colpi di click nella Console OCI, rimappata da un indirizzo non-`for_each` `oci_core_instance.instance` tramite un blocco `moved`:

```hcl
moved {
  from = oci_core_instance.instance
  to   = oci_core_instance.node["instance-20260402-1401"]
}
```

Quella singola riga è la differenza tra "refactor verso `for_each`" e "distruggi e ricrea l'unico nodo che il cluster ha." Scalare verso l'HA o aggiungere worker significa solo aggiungere voci a `k3s_nodes` con `role = "agent"` e un `fault_domain` distribuito su `FAULT-DOMAIN-1/2/3` — questa regione ha un solo availability domain, quindi i fault domain sono l'unico asse di ridondanza disponibile. Oracle ha dimezzato recentemente l'allowance Always Free su Ampere il 15/06/2026, passando da 4 OCPU / 24GB a 2 OCPU / 12GB per l'intera tenancy. L'unico nodo qui è già dimensionato a 2 OCPU / 12GB, quindi da solo consuma già *l'intera* allowance gratuita.

---

## Networking: Una Security List Condivisa, Non Regole Per Istanza

VCN, subnet, DHCP options, route table e la security list di default sono tutte gestite tramite `manage_default_resource_id` invece che come risorse custom separate — il pattern che la Console OCI lascia quando modifichi i default invece di crearne di nuovi. Ci sono anche due NSG (`nsg_quick_action_1`/`_2`) che sono puri residui della console: solo egress, nessuna regola di ingress, artefatti di aver cliccato "Quick Actions" nella UI agli inizi.

Tutte le regole di ingress reali vivono sull'unica security list di default, condivisa da tutta la subnet:

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

ssh, https, l'API server di k3s (6443), kubelet (10250), il VXLAN di flannel (8472/udp), node-exporter (9100) ed etcd peer/client (2379-2380) hanno ciascuno il proprio blocco, con lo scope limitato al CIDR della VCN dove la porta non ha motivo di essere raggiungibile da internet. Poiché è una lista condivisa e non security group per istanza, un nuovo nodo aggiunto a `k3s_nodes` è coperto automaticamente da ogni regola esistente — nessun collegamento NSG per istanza.

Una trappola non ovvia: `ingress_security_rules` è un `set` Terraform/OpenTofu, e il suo campo `description` è opzionale-ma-computed. Lasciare `description` non impostato su una regola *nuova* mentre ogni regola esistente ne ha già una fa sì che `tofu plan` mostri ogni altra regola come rimossa-e-riaggiunta — un artefatto cosmetico del diff sull'set, non una modifica reale, ma allarmante la prima volta che vedi un plan sostenere di toccare sette regole che non hai chiesto. Impostare `description` esplicitamente su ogni regola, sempre, evita il problema del tutto.

---

## IAM: Un Dynamic Group Che Si Costruisce Da Solo

L'istanza deve leggere segreti da un vault senza che nessuna credenziale statica sia incorporata da nessuna parte. OCI risolve questo tramite i **dynamic group** — un gruppo la cui membership viene calcolata da una matching rule sui metadati dell'istanza, invece che assegnata a mano — abbinati a policy che concedono al gruppo permessi specifici.

La matching rule viene generata dalla stessa mappa `for_each` delle istanze, quindi copre sempre esattamente i nodi che esistono:

```hcl
resource "oci_identity_dynamic_group" "dynamic_group" {
  matching_rule = "ANY {${join(", ", [
    for k, n in oci_core_instance.node : "instance.id = '${n.id}'"
  ])}}"
}
```

Aggiungi un nodo a `k3s_nodes`, e finisce automaticamente nel dynamic group al prossimo apply — nessun collegamento identity separato per nodo. Due policy si appoggiano su di esso: una concede accesso in lettura a segreti e vault per l'intera tenancy (`Allow dynamic-group '.../dynamic-group' to read secret-family in tenancy`), l'altra limita l'accesso in scrittura esattamente al bucket di backup (`where target.bucket.name='...'`). Una terza policy, non correlata, concede al gruppo `Administrators` i permessi di tenant-admin — una policy fissa ereditata dal setup via Console, non qualcosa derivato dal grafo dei nodi.

Questo dynamic group è esattamente il meccanismo che il [post sull'homelab](/posts/homelab/) liquida con "autenticandosi con l'identità cloud propria dell'istanza compute anziché con una credenziale archiviata nel cluster." Il `ClusterSecretStore` su k3s con `principalType: InstancePrincipal` legge proprio attraverso questa policy.

---

## Il Vault e il Bucket di Backup

Un piccolo vault KMS e una chiave AES-256 protetta via software sono ciò che sta dietro i segreti che il nodo legge tramite quel dynamic group:

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

Accanto ad esso, un bucket Object Storage (`NoPublicAccess`, senza versioning, tier Standard) esiste puramente come destinazione di backup — dove CNPG invia gli snapshot, e dove il datastore etcd embedded di k3s può puntare direttamente tramite il suo endpoint S3-compatibile. Questo secondo percorso merita una menzione a parte: l'upload degli snapshot di etcd non capisce gli instance principal, solo chiavi di accesso/segreto statiche — una "Customer Secret Key" OCI generata sotto un utente, non una API key e non il dynamic group di cui sopra. È l'unico punto di tutto questo setup dove serve ancora una credenziale ferma in un file di configurazione invece di un'identità.

---

## Ansible: Patching e Bootstrap, Deliberatamente Separati da Tofu

k3s in sé non fa parte dello state o del resource graph di OpenTofu — cloud-init non lo installa mai, e non c'è nessun blocco di provisioning `oci_core_instance` per lui. Viene installato e aggiornato interamente tramite Ansible, contro un inventory che gli output di Tofu aiutano a compilare a mano (l'output `k3s_nodes` fornisce role/public_ip/private_ip per nodo) ma non generano automaticamente.

Due playbook, stesso inventory, due compiti diversi:

```bash
# Manutenzione a livello OS — apt upgrade, autoremove, autoclean, reboot condizionale
ansible-playbook -i inventory.yml update.yml

# (Re-)installa k3s tramite la collection esterna k3s-ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory.yml bootstrap-k3s.yml
```

`bootstrap-k3s.yml` è un wrapper di una riga che importa `k3s.orchestration.site` dalla collection [k3s-ansible](https://github.com/k3s-io/k3s-ansible), pinnata su un git ref in `requirements.yml`. La configurazione vera vive nelle vars del gruppo `k3s_cluster` in `inventory.yml`, e due di esse sono spigoli vivi:

- **`token`** deve corrispondere a quello già presente sul nodo (`/var/lib/rancher/k3s/server/token`), altrimenti un run reale riscrive silenziosamente il join token del cluster.
- **`server_config_yaml`** deve rispecchiare nell'intento, byte per byte, il `/etc/rancher/k3s/config.yaml` già esistente sul nodo, perché un run reale sovrascrive sempre quel file a partire da questa variabile — tutto ciò che è configurato sul nodo ma manca da `server_config_yaml` (come `disable: [traefik]`) viene silenziosamente eliminato, non unito.

`k3s_version` è un bump deliberato e manuale — rieseguire il playbook con un pin più recente innesca un vero upgrade su un cluster single-node live, quindi non è qualcosa da lasciare a deriva.

---

## Cosa Viene Fatto il Provisioning

| Livello | Strumento | Cosa |
| --- | --- | --- |
| Compute | OpenTofu | Istanza/e ARM `A1.Flex`, `for_each` su `k3s_nodes` |
| Rete | OpenTofu | VCN, subnet, security list di default condivisa, due NSG residue |
| IAM | OpenTofu | Dynamic group auto-costruito, policy per segreti/vault/bucket di backup |
| Segreti | OpenTofu | Vault KMS + chiave AES-256 |
| Backup | OpenTofu | Bucket Object Storage (backup) |
| Manutenzione OS | Ansible | apt upgrade/autoremove/autoclean, reboot condizionale |
| Ciclo di vita k3s | Ansible | Collection k3s-ansible, install/upgrade con versione pinnata |

---

## Conclusione

Nulla di tutto questo è entusiasmante di per sé — una VM, una VCN, un vault, un paio di policy. Ciò che vale la pena raccontare è il punto di giunzione: il compito di OpenTofu finisce nel momento in cui l'istanza e la sua identità esistono, il compito di Ansible finisce nel momento in cui k3s è in esecuzione e unito al cluster, e tutto ciò che è descritto nel [post sull'homelab](/posts/homelab/) — Flux, Keycloak, il vault esterno via instance principal — parte da lì. Tre strumenti, tre responsabilità ristrette, e un passaggio di consegne pulito tra loro è un adattamento migliore per un homelab a un nodo rispetto a un unico strumento che prova a possedere l'intero ciclo di vita.
