---
title: "Self-Hosting di GitLab CE su Kubernetes con Helmfile"
date: 2026-06-21
draft: false
tags: ["kubernetes", "gitlab", "helmfile", "envoy-gateway", "cloudnative-pg", "rustfs"]
description: "Un deployment production-grade di GitLab CE su Kubernetes con Helmfile, Envoy Gateway, CloudNative-PG e RustFS."
ShowToc: true
---

Fare self-hosting di GitLab è un'idea abbastanza comune. Farlo bene — con TLS corretto, un vero operatore per il database, object storage S3-compatibile e una gestione dei segreti che non implichi incollare credenziali nello YAML — è dove la maggior parte delle guide si ferma. Questo articolo illustra un deployment di GitLab CE su Kubernetes basato su Helmfile che prende sul serio questi aspetti.

---

## Perché Helmfile

Il chart Helm di GitLab ha molti componenti mobili: dipende da un database PostgreSQL esterno, una cache Redis-compatibile esterna e un object store S3-compatibile. Gestire queste dipendenze — e assicurarsi che esistano prima che il chart GitLab tenti di usarle — è esattamente il punto di forza di Helmfile.

Helmfile orchestra più release Helm in ordine di dipendenza. Ogni release dichiara i propri `needs`, e Helmfile garantisce che le dipendenze siano distribuite e pronte prima di procedere con quelle che dipendono da esse. Per uno stack come questo, quell'ordinamento è fondamentale.

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

## Ingress: Envoy Gateway e Gateway API

Al posto di nginx-ingress, questo setup usa **Envoy Gateway** come controller Gateway API. La Gateway API è il successore moderno dell'`Ingress` Kubernetes — più espressiva, con una migliore separazione dei ruoli, e verso cui l'ecosistema si sta chiaramente orientando.

La `GatewayClass` e il `Gateway` vengono creati entrambi in un hook `postsync` immediatamente dopo l'installazione di Envoy Gateway — prima che cert-manager giri. Questo ordinamento è intenzionale: cert-manager ha bisogno che il Gateway esista per poter eseguire la challenge HTTP-01 ACME.

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

I nomi dei listener (`gitlab-web`, `registry-web`) corrispondono ai valori `sectionName` che il chart GitLab usa nei `parentRefs` delle HTTPRoute — un requisito non ovvio.

Il chart GitLab è configurato con `global.gatewayApi.gatewayRef` che punta a questo Gateway. Con questa impostazione, il chart salta la creazione della propria risorsa Gateway e crea solo HTTPRoute. Senza di essa, Helm tenterebbe di rivendicare la proprietà di una risorsa esistente che non ha creato e fallirebbe.

RustFS è interno al cluster e non è esposto tramite il Gateway.

---

## TLS: cert-manager e Let's Encrypt

[cert-manager](https://cert-manager.io) gestisce il TLS. Per la produzione, un `ClusterIssuer` è configurato per usare Let's Encrypt ACME con la challenge HTTP-01 tramite il solver `gatewayHTTPRoute` di cert-manager. Il solver crea una HTTPRoute temporanea sul Gateway `gitlab-gw` per ogni challenge — motivo per cui il Gateway deve esistere prima che cert-manager giri.

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

Invece di un certificato wildcard, il cert elenca i SAN espliciti: `gitlab.thewalterman.cloud` e `registry.thewalterman.cloud`. HTTP-01 non può validare domini wildcard, quindi ogni hostname va elencato esplicitamente.

L'hook `postsync` di cert-manager crea le risorse `ClusterIssuer` e `Certificate`, poi attende che il certificato diventi `Ready` prima di ritornare. Questo significa che il secret TLS esiste prima che il chart GitLab si installi.


---

## Database: CloudNative-PG

Il chart GitLab non include PostgreSQL — si aspetta un database esterno. [CloudNative-PG](https://cloudnative-pg.io) (CNPG) funziona come operatore in `cnpg-system` e gestisce PostgreSQL come risorsa Kubernetes nativa.

La release `cnpg-gitlab` distribuisce un oggetto `Cluster`. CNPG gestisce l'intero ciclo di vita: elezione del primary, connection pooling e health check. GitLab si connette tramite `gitlab-rails-db-rw.gitlab.svc.cluster.local` — il servizio read-write che CNPG mantiene puntato al primary corrente.

La catena di dipendenze in Helmfile garantisce che CNPG sia operativo prima che l'oggetto cluster venga applicato, e che il cluster sia in salute prima che GitLab si avvii.

---

## Backup del Database

L'integrazione dei backup di CNPG è configurata interamente tramite i valori Helm. La release `cnpg-gitlab` abilita l'archiviazione continua dei WAL e i backup base giornalieri via barman, puntando all'endpoint interno di RustFS:

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

Le credenziali di backup (`cnpg-backup-s3`) vengono create da un hook `presync` sulla release `cnpg-gitlab`, leggendo dallo stesso file `.env` delle altre variabili. `secret.create: false` indica al chart di non tentare di crearlo — l'hook lo ha già fatto.

Questo fornisce l'archiviazione WAL dal momento in cui il cluster si avvia, più un backup base giornaliero, con una retention di 7 giorni — il tutto senza nessuna configurazione manuale dopo `helmfile sync`.

---

## Cache: Valkey

[Valkey](https://valkey.io) è la cache Redis-compatibile che GitLab utilizza per sessioni, code sidekiq e dati di cache. È un fork community di Redis mantenuto sotto licenza open source — funzionalmente identico dal punto di vista di GitLab.

Il secret della password viene creato da un hook `presync` di Helmfile prima dell'installazione del chart:

```bash
kubectl create secret generic valkey-auth \
  --namespace gitlab \
  --from-literal=default="${VALKEY_PASSWORD:?}" \
  --dry-run=client -o yaml | kubectl apply -f -
```

La sintassi `${VALKEY_PASSWORD:?}` fallisce in modo esplicito se la variabile non è impostata, intercettando l'errore comune di eseguire `helmfile sync` senza prima aver fatto `source .env`. La password non viene mai scritta in nessun file tracciato da Git — vive in `.env` (ignorato da git) e viene materializzata come Kubernetes Secret al momento del deploy.

---

## Object Storage: RustFS

Il chart GitLab normalmente include MinIO, ma questo setup lo sostituisce con [RustFS](https://rustfs.com) — un'implementazione S3 compatibile con MinIO. Espone la stessa S3 API, il che significa che la configurazione del chart GitLab rimane invariata; si punta semplicemente a un endpoint diverso.

Le credenziali non vengono mai inserite nel file dei valori. Al contrario, un hook `presync` crea il Kubernetes Secret `rustfs-auth` dalle variabili d'ambiente della shell, e il chart vi fa riferimento tramite `secret.existingSecret`:

```yaml
secret:
  existingSecret: rustfs-auth
```

Lo stesso hook crea anche i secret necessari al chart GitLab (`gitlab-object-storage`, `gitlab-registry-storage`) per connettersi a RustFS.

Il provisioning dei bucket è gestito da un Kubernetes `Job` negli `extraManifests` del chart. Viene eseguito all'installazione e all'upgrade, creando tutti i bucket che GitLab si aspetta — artifacts, uploads, lfs-objects, packages, registry e altri, incluso il bucket `cnpg-backup` usato per i backup del database. Il Job legge le credenziali da `rustfs-auth` tramite `envFrom`, quindi nessuna credenziale appare nella specifica del Job.

RustFS è raggiungibile solo all'interno del cluster. GitLab e CNPG vi accedono tramite `http://rustfs-svc.gitlab.svc.cluster.local:9000` — non esiste un listener Gateway esterno per S3.

---

## Flusso di Deployment

Setup iniziale:

```bash
mise trust && mise install          # installa helm, helmfile, k3d, kubectl
mise run env-init                   # genera .env con credenziali casuali
k3d cluster create -c k3d.yaml     # crea il cluster locale
source .env && helmfile sync        # distribuisce lo stack completo
```

Operatività quotidiana:

```bash
source .env && helmfile diff        # anteprima modifiche rispetto al cluster live
source .env && helmfile sync        # applica tutte le release
```

Il `source .env` è necessario perché gli hook di helmfile leggono le credenziali dalle variabili d'ambiente della shell. Dimenticarlo produce un errore esplicito (`${VAR:?}` fallisce) invece di una configurazione silenziosamente errata.

---

## Cosa Gira

Dopo un deploy completato con successo, i seguenti servizi sono accessibili con TLS valido:

| Servizio | URL |
| ------- | --- |
| GitLab | `https://gitlab.thewalterman.cloud` |
| Container Registry | `https://registry.thewalterman.cloud` |

---

## A Cosa Serve

Il caso d'uso ovvio è sviluppare e testare pipeline GitLab CI in locale — si ottiene un GitLab reale con un runner reale, artifact storage reale e un container registry, senza dover puntare le pipeline a un ambiente condiviso.

È anche un riferimento utile per chiunque voglia distribuire GitLab su Kubernetes al di fuori delle offerte cloud gestite: il modello di dipendenze di Helmfile, il pattern degli hook presync per i segreti (mai nello YAML tracciato), la sostituzione di MinIO con RustFS e l'integrazione dei backup CNPG sono tutti direttamente trasferibili a deployment in produzione.

L'intero setup gira su un singolo nodo k3d con un utilizzo di risorse ragionevole, rendendolo pratico da usare in parallelo con altro lavoro di sviluppo.
