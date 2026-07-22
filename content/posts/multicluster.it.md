---
title: "Gestire Più Cluster Kubernetes con FluxCD e il Modello Hub-Spoke"
date: 2026-03-30
draft: false
tags: ["kubernetes", "gitops", "flux", "multi-cluster", "hub-spoke"]
description: "Come gestire più cluster Kubernetes da un unico repository Git usando FluxCD e una topologia hub-spoke."
ShowToc: true
---

Gestire un singolo cluster Kubernetes è già abbastanza impegnativo. Gestirne diversi — mantenendoli coerenti, eseguendo deploy in sicurezza e senza impazzire nel processo — è dove le cose si fanno interessanti. Questo articolo illustra un setup GitOps che gestisce cluster Kubernetes di staging e produzione da un unico repository Git usando FluxCD e una topologia hub-spoke.

---

## Il Problema della Gestione Multi-Cluster

La maggior parte dei team inizia con un solo cluster. Poi aggiunge un ambiente di staging. Poi un ambiente di produzione in una regione diversa. Prima che te ne accorga, ti ritrovi con un sacco di script `kubectl apply`, pipeline CI raffazzonate e la sensazione che staging e produzione si siano silenziosamente divisi.

La risposta standard del GitOps è: Git è la fonte di verità, e un controller riconcilia continuamente il cluster verso quello stato. Ma quando hai *più* cluster, emerge una nuova domanda: dove gira il controller?

Un'opzione è eseguire Flux su ogni cluster in modo indipendente. Funziona, ma significa gestire Flux stesso su ogni cluster, e non c'è un unico posto dove osservare cosa sta succedendo in tutta la flotta.

Un'opzione più pulita è il **modello hub-spoke**: un cluster hub dedicato esegue Flux ed è responsabile della riconciliazione dei manifest su tutti i cluster remoti.

---

## Architettura

```sh
GitHub (fonte di verità)
  └─ Flux sul cluster hub
       ├─ hub/production.yaml
       └─ hub/staging.yaml
```

In totale ci sono tre cluster Kubernetes:

- **Hub** — esegue Flux e nient'altro di sostanziale. È il piano di controllo della flotta.
- **Staging** — riceve i workload riconciliati dall'hub.
- **Production** — uguale, ma produzione.

---

## Bootstrap di Flux con flux-operator

Invece di usare `flux bootstrap github` (che committa manifest generati automaticamente nel repository), il cluster hub viene bootstrappato con [flux-operator](https://fluxoperator.dev/). L'operator viene installato una volta tramite Helm, e una singola custom resource `FluxInstance` dichiara la distribuzione Flux desiderata e la configurazione di sync del GitRepository. Da quel momento l'operator gestisce il ciclo di vita dei controller Flux.

Questo mantiene il repository pulito — nessun file `gotk-components.yaml` auto-generato da un comando di bootstrap — e gli aggiornamenti a Flux stesso diventano una modifica di una riga a `spec.distribution.version` nel `FluxInstance`.

---

## Come Flux Raggiunge i Cluster Remoti

Flux sull'hub ha bisogno di credenziali per comunicare con i server API dei cluster remoti. Queste sono memorizzate come secret Kubernetes sull'hub stesso — un secret kubeconfig per cluster remoto, in un namespace dedicato:

```sh
cluster hub
  ├── namespace: staging
  │     └── secret: cluster-kubeconfig
  └── namespace: production
        └── secret: cluster-kubeconfig
```

Ogni risorsa `Kustomization` di Flux sull'hub fa riferimento al kubeconfig del proprio cluster remoto tramite `spec.kubeConfig.secretRef`. Quando Flux riconcilia quella Kustomization, applica i manifest al cluster remoto invece che all'hub.

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

## Fasi di Deploy e Dipendenze

Ogni cluster remoto riceve quattro fasi di configurazione. La catena di dipendenze è: tenants → infrastructure → {configurations e apps in parallelo}. Flux attende che ogni fase sia pronta prima di avviare quelle che dipendono da essa.

**1. Tenants** — Crea il namespace, un ServiceAccount `flux-cluster-admin` e un ClusterRoleBinding che gli assegna il ruolo cluster-admin. Questa è l'identità che Flux utilizzerà quando applicherà le fasi successive al cluster remoto.

**2. Infrastructure** — Distribuisce [Envoy Gateway](https://gateway.envoyproxy.io/) tramite HelmRelease. Poiché Helm deve comunicare con il cluster remoto, ogni HelmRelease viene patchato con `spec.kubeConfig` e `spec.serviceAccountName` al momento della riconciliazione.

**3. Configurations** — Crea le risorse `GatewayClass`, `Gateway` e `HTTPRoute` (Kubernetes Gateway API) necessarie per instradare il traffico nel cluster.

**4. Apps** — Distribuisce le applicazioni come HelmRelease, con valori specifici per ambiente patchati per cluster.

Il grafo delle dipendenze garantisce che non si possa mai finire con le app distribuite prima che l'infrastruttura sia pronta.

---

## Mantenere Staging e Production DRY

La directory `deploy/` contiene le definizioni base condivise — gli HelmRelease di infrastruttura, i template RBAC per i tenant, l'HelmRepository e l'HelmRelease base delle app. `clusters/staging/` e `clusters/production/` sono overlay Kustomize che fanno riferimento a queste basi e applicano patch specifiche per ambiente.

Ad esempio, la definizione base dell'app [homepage](https://gethomepage.dev/) si trova in `deploy/apps/homepage.yaml`. L'overlay di staging in `clusters/staging/apps/homepage.yaml` applica i valori specifici per staging:

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

La Kustomization a livello hub per staging poi patcha *tutti* gli HelmRelease in quel percorso con il riferimento al kubeconfig remoto, così non devi ripeterlo per ogni app:

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

Questo approccio a strati — definizioni base, overlay per cluster, patch a livello hub — mantiene lo YAML minimale ed evita il copia-incolla tra ambienti.

---

## Cosa Ottieni

Il modello hub-spoke con Flux ti offre alcune cose difficili da ottenere altrimenti:

- **Pannello di controllo unico** — tutti gli eventi di riconciliazione sono visibili dall'hub. Un solo `kubectl --context k3d-hub get kustomizations -A` mostra lo stato di ogni cluster.
- **Tooling coerente** — staging e production usano la stessa versione di Flux e le stesse basi Kustomize. Le differenze sono patch esplicite, non drift non documentato.
- **Git come audit log** — ogni modifica a qualsiasi cluster passa attraverso una pull request. Il diff è esattamente ciò che verrà applicato.
- **Ordinamento delle dipendenze** — il `dependsOn` di Flux garantisce che i cluster facciano bootstrap nell'ordine corretto ogni volta, che si tratti della prima installazione o del recupero da un guasto.

Per i team che gestiscono più di un cluster, questo pattern scala naturalmente: aggiungere un nuovo ambiente significa aggiungere una nuova directory overlay e una nuova Kustomization sull'hub, senza toccare nessuna configurazione esistente.
