---
title: "Costruire un Homelab Production-Grade con Kubernetes, GitOps e Sicurezza Zero-Trust"
date: 2026-03-27
draft: false
tags: ["kubernetes", "gitops", "homelab", "security", "flux"]
description: "Come costruire un homelab production-grade con uno stack applicativo completamente protetto da OIDC usando GitOps e principi zero-trust."
ShowToc: true
---

Gestire un homelab nel 2026 non significa più avere un server polveroso sotto la scrivania con un IP statico e un certificato auto-firmato che devi ignorare ogni volta. Gli homelab moderni possono — e dovrebbero — essere costruiti con lo stesso rigore dell'infrastruttura di produzione. Questa è la storia esattamente di questo: un homelab Kubernetes alimentato da FluxCD, OpenBao, Keycloak e uno stack di applicazioni completamente protetto da OpenIDConnect, distribuito su due ambienti con automazione GitOps completa.

Il risultato è una piattaforma in cui:

- Ogni modifica è un commit Git
- I segreti non vivono mai nei file YAML
- Ogni applicazione si autentica tramite un provider di identità centralizzato
- Il TLS è completamente automatizzato
- Lo sviluppo locale rispecchia la produzione il più fedelmente possibile

Vediamo nel dettaglio come funziona tutto.

---

## Il Quadro Generale

L'infrastruttura punta a due ambienti cluster gestiti da un unico repository Git:

- **k3d** — un cluster di sviluppo locale che gira in Docker, con certificati auto-firmati e domini `.homelab.localhost`
- **k3s** — un cluster di produzione bare-metal/VM che usa certificati Let's Encrypt (tramite validazione DNS-01) e un dominio reale

[k3s](https://k3s.io) è una distribuzione Kubernetes leggera — un singolo binario privo delle integrazioni cloud-provider, progettato per deployment bare-metal ed edge. [k3d](https://k3d.io) avvolge k3s in container Docker, rendendo possibile eseguire la stessa distribuzione in locale senza VM. Poiché entrambi i target usano lo stesso runtime k3s, il comportamento di rete, il ciclo di vita dei container e la compatibilità API sono coerenti — non c'è una distribuzione separata per il dev locale di cui tener conto.

[FluxCD](https://fluxcd.io) è il motore GitOps: osserva il repository e riconcilia continuamente lo stato del cluster con ciò che è dichiarato in Git. Non esiste `kubectl apply` nelle operazioni quotidiane. Ogni risorsa è versionata, revisionabile e verificabile.

Il repository è organizzato in cinque sezioni principali:

```sh
clusters/        # Entry point Flux Kustomization e FluxInstance per target
infrastructure/  # Envoy Gateway, cert-manager, External Secrets Operator
apps/            # Keycloak, OpenBao, CNPG, RustFS
monitoring/      # Prometheus, Grafana, Loki, Fluent Bit
configurations/  # Gateway, HTTPRoute, Issuer, ExternalSecret, PodMonitor
```

Ogni sezione segue lo stesso schema di overlay:

```sh
<sezione>/
  base/   # Risorse indipendenti dall'ambiente (Helm release, configurazioni base)
  k3d/    # Patch e override per lo sviluppo locale
  k3s/    # Patch e override per la produzione
```

La struttura `base/k3d/k3s` mantiene la configurazione DRY senza ramificare in repository separati per cluster. `base/` contiene tutto ciò che è indipendente dall'ambiente — la specifica completa della Helm release, Namespace, HTTPRoute ed ExternalSecret. `k3d/` e `k3s/` contengono solo ciò che differisce: patch per hostname, riferimenti all'issuer TLS e valori Helm specifici per ambiente. I diff restano piccoli e verificabili — una modifica in produzione è esattamente un file di patch, non un diff spalmato su configurazioni duplicate.

Un `ResourceSet` Flux in `configurations/base/` si auto-gestisce la `HelmRelease` e l'`OCIRepository` del flux-operator, mantenendo l'operatore aggiornato senza intervento manuale.

### Fedeltà nell'Ambiente Locale

Il cluster k3d è progettato per rispecchiare la topologia di produzione il più fedelmente possibile senza accesso a internet. Tutti i task di setup vengono eseguiti tramite [mise](https://mise.jdx.dev), che gestisce le versioni degli strumenti e carica le variabili d'ambiente. Tre dettagli non ovvi rendono il dev locale funzionante correttamente:

- **mkcert CA**: `mise run k3d-create` genera una CA locale con [mkcert](https://github.com/FiloSottile/mkcert), installata sull'host e importata come TLS Secret così l'issuer cert-manager di k3d può firmarci il certificato wildcard. Ogni pod che esegue una propria chiamata di OIDC discovery verso Keycloak (Grafana, ad esempio) monta la stessa CA nel proprio trust store tramite un secret volume aggiuntivo — senza di essa, la validazione TLS dell'issuer URL auto-firmato fallisce.
- **hostAliases**: `k3d.yaml` mappa `keycloak.homelab.localhost → 172.28.0.1` (il gateway del bridge Docker). Senza questa configurazione, i pod non possono risolvere l'URL Keycloak usato per l'OIDC discovery, poiché quell'hostname è risolvibile solo dall'host.
- **Registry cache**: i pull da Docker Hub vengono proxati attraverso una cache locale (`docker-io:5000`), evitando i limiti di rate durante lo sviluppo iterativo.

---

## Networking: Gateway API con Envoy Gateway

Invece del classico `Ingress` di Kubernetes, questo setup usa la **Gateway API** — il successore moderno e più espressivo che offre una migliore separazione dei ruoli e semantiche di routing più ricche.

[Envoy Gateway](https://gateway.envoyproxy.io/) funge da **controller GatewayClass**, e una singola risorsa `Gateway` gestisce tutto il traffico HTTPS del cluster. In k3d, il Gateway ascolta su `*.homelab.localhost` quindi non è necessario modificare `/etc/hosts`.

Ogni applicazione riceve un `HTTPRoute` che punta al suo servizio backend:

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

Pulito, dichiarativo e con scope di namespace — esattamente come la Gateway API è stata progettata.

---

## Rate Limiting e Rilevamento dell'IP Client

Le CRD di policy di Envoy Gateway aggiungono controlli a livello HTTP al di sopra del modello di routing della Gateway API.

**BackendTrafficPolicy** applica limiti di frequenza per `HTTPRoute`. Gli endpoint di autenticazione di Keycloak sono i candidati naturali: il token endpoint (`/realms/apps/protocol/openid-connect/token`) è limitato a 20 richieste/minuto per IP sorgente distinto, il login endpoint (`/realms/apps/login-actions/authenticate`) a 10 richieste/minuto. Il tipo di policy è `Local`, ovvero i limiti vengono applicati per istanza Envoy e non condivisi su un fleet — sufficiente per un homelab single-node.

Perché il rate limiting funzioni correttamente dietro proxy, Envoy Gateway deve conoscere l'IP reale del client — non l'IP proxy. **ClientTrafficPolicy** risolve questo a livello di Gateway.

Il reverse proxy imposta `CF-Connecting-IP` sull'IP originale del client su ogni richiesta proxiata e rimuove qualsiasi versione inviata dal client — quindi non può essere falsificato. Con questa policy attiva, Envoy Gateway usa quell'header come IP sorgente autorevole per il rate limiting, l'access logging e tutto ciò che dipende dal sapere chi sta effettivamente facendo la richiesta.

---

## TLS: Completamente Automatizzato, Zero Passaggi Manuali

[cert-manager](https://cert-manager.io) gestisce il ciclo di vita dei certificati in entrambi gli ambienti. Entrambi gli overlay definiscono un `ClusterIssuer` chiamato `ci-issuer` — stesso nome, tipo diverso — così l'annotazione del `Gateway` base `cert-manager.io/cluster-issuer: ci-issuer` funziona senza modifiche in entrambi gli ambienti senza bisogno di patch.

**k3d (sviluppo locale)**:

`mise run k3d-create` genera una CA locale con mkcert e `mise run flux-bootstrap k3d` la importa come TLS Secret nel namespace cert-manager. L'overlay k3d definisce un CA issuer basato su quel secret. cert-manager emette poi un certificato wildcard per `*.homelab.localhost`, che il Gateway usa per la terminazione TLS. Poiché la stessa CA è attendibile sia dall'host (mkcert la installa nel system store) che dal cluster (montata al bootstrap), browser e validazione OIDC in-cluster funzionano entrambi senza errori.

**k3s (produzione)**:

L'overlay k3s definisce lo stesso issuer come ACME issuer. Il token del provider DNS è archiviato in un vault esterno, recuperato in un Kubernetes Secret da External Secrets Operator e passato a cert-manager per la validazione DNS-01. L'intero flusso di provisioning è automatizzato e privo di segreti dal punto di vista di Git.

---

## Gestione dei Segreti: Due Backend, un Solo External Secrets Operator

Questa è la parte architetturalmente più interessante dello stack, e dove viene applicato il principio "zero segreti in Git". Entrambi i target cluster si affidano allo stesso schema — un `ClusterSecretStore` chiamato `secret-store` che alimenta l'[External Secrets Operator](https://external-secrets.io) — ma ciascuno punta a un backend diverso:

```
(OpenBao su k3d | un vault esterno su k3s) → External Secrets Operator → Kubernetes Secret → pod applicativi
```

Poiché il nome del `ClusterSecretStore` e le chiavi dei segreti sono identici su entrambi i target, ogni manifest `ExternalSecret` in `base/` è condiviso senza modifiche tra k3d e k3s — nessuna patch per target necessaria per il consumo dei segreti, solo per quale backend punta effettivamente `secret-store`.

### OpenBao (k3d)

[OpenBao](https://openbao.org) è un fork di HashiCorp Vault mantenuto dalla community (a seguito del cambio di licenza). Su k3d gira in modalità alta disponibilità con storage Raft, distribuito tramite Helm. Dopo l'inizializzazione, le chiavi di unseal vengono archiviate come Kubernetes Secret in modo che un hook post-start possa sbloccare automaticamente il pod al riavvio.

Il ciclo di vita di OpenBao è gestito tramite task `mise`:

```bash
mise run bao-init     # Inizializza, sblocca e popola i segreti
```

Su k3d, `secret-store` si autentica a OpenBao usando il metodo di autenticazione Kubernetes — il ruolo `apps` è vincolato al ServiceAccount `eso-reader`:

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

### Vault Esterno (k3s)

In produzione OpenBao non gira affatto — non aveva senso gestire un Vault self-managed e stateful su una macchina di produzione single-node quando era già disponibile un vault gestito. Su k3s, `secret-store` punta invece a quel vault esterno tramite il provider corrispondente di ESO, autenticandosi con l'identità cloud propria dell'istanza compute anziché con una credenziale archiviata nel cluster:

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

L'identità dell'istanza riceve accesso in lettura al secret store direttamente tramite l'IAM del provider — nessuna credenziale statica, nessun bootstrap di segreti in-cluster. L'ID del vault e la region non sono mai committati in Git: vengono iniettati tramite `postBuild.substituteFrom` di Flux sulla Kustomization `configurations`, leggendo `${VAULT_ID}`/`${VAULT_REGION}` da un Secret creato manualmente in `flux-system`. I segreti stessi vengono seminati direttamente tramite la console/CLI del provider del vault — non esiste un task di automazione per questo, deliberatamente, trattandosi di un passaggio di setup una tantum fuori dal loop GitOps.

### External Secrets Operator

Ogni namespace ha una risorsa `ExternalSecret` dedicata che recupera i segreti rilevanti da `secret-store` e li materializza come Kubernetes Secret. Questo significa che nessuna applicazione legge mai direttamente da OpenBao o dal vault esterno — consuma semplicemente un Kubernetes Secret standard. L'ExternalSecret si aggiorna ogni minuto, quindi la rotation è quasi in tempo reale, su entrambi i target.

---

## Object Storage: RustFS su k3d, un Bucket Esterno su k3s

L'object storage segue una versione più blanda della stessa divisione usata per i segreti: k3d fa girare un proprio store S3-compatibile in-cluster, mentre k3s punta semplicemente a un bucket che esiste già altrove. Qui non c'è un'astrazione a livello Kubernetes equivalente a `secret-store` — ogni consumer (il plugin di backup di CNPG, l'agente di snapshot di OpenBao) parla già nativamente l'API S3, quindi cambiare backend è solo questione di un endpoint e una coppia di credenziali diversi, non una CRD.

### RustFS (k3d)

[RustFS](https://rustfs.com) è un object store compatibile con l'API MinIO, distribuito come singola istanza standalone tramite Helm. Si autentica tramite Keycloak via OIDC — stesso pattern di ogni altra applicazione dello stack, con le credenziali del client recuperate tramite un `ExternalSecret` — e la sua web console è raggiungibile tramite un proprio `HTTPRoute`.

Il bootstrap dei bucket è un `Job` Kubernetes one-shot che esegue il client `mc` contro RustFS subito dopo l'avvio: crea i bucket `keycloak-backup` e `openbao-backup` e una policy di accesso admin, usando credenziali S3 che ESO ha già recuperato da OpenBao. Non è circolare — sono chiavi di accesso statiche, non qualcosa che RustFS stesso deve prima fare il backup.

### Bucket Esterno (k3s)

k3s non fa girare alcun workload di object storage — `apps/k3s/kustomization.yaml` distribuisce solo `cnpg` e `keycloak`, nulla legato allo storage. Il plugin barman-cloud di CNPG parla invece direttamente con un bucket S3-compatibile esterno, con endpoint e percorso di destinazione risolti dallo stesso meccanismo `postBuild.substituteFrom` di Flux usato per i dettagli di connessione al vault, leggendo da un Secret `objectstore-substitutions` in `flux-system`. Anche qui non c'è nessun job di snapshot di OpenBao di cui preoccuparsi, dato che OpenBao stesso non gira su k3s.

---

## Identità: Keycloak come Provider OIDC Centralizzato

[Keycloak](https://www.keycloak.org) gira nel suo namespace supportato da un'istanza PostgreSQL gestita da **CloudNativePG** (CNPG). Funge da emittente OIDC per ogni applicazione nello stack:

| Applicazione | Meccanismo di autenticazione |
| --- | --- |
| Grafana | `generic_oauth` (OIDC) tramite Helm values |
| OpenBao UI | Metodo OIDC configurato in OpenBao — solo k3d |
| RustFS | Login OIDC via Keycloak (credenziali da ExternalSecret) — solo k3d |

Il mapping dei ruoli è centralizzato: se un utente appartiene al gruppo `admins` in Keycloak, ottiene l'accesso admin.

Una volta che Keycloak è pronto, puoi configurare i clients OIDC:

```bash
mise run config-oidc   # scrive i client secret OIDC di Grafana e RustFS in OpenBao
```

---

## Database: CloudNativePG

[CloudNativePG](https://cloudnative-pg.io) (CNPG) è un operatore Kubernetes per PostgreSQL che gestisce l'intero ciclo di vita del database come risorse native. Invece di uno StatefulSet con replica manuale, si dichiara un oggetto `Cluster` e CNPG gestisce automaticamente l'elezione del primario, la replica in streaming e il failover.

Il database di Keycloak è un `Cluster` CNPG definito in `apps/base/keycloak/`. La HelmRelease di Keycloak ha `dependsOn: cnpg`, quindi non tenta mai di avviarsi prima che l'operatore del database sia pronto. Questo ordinamento è fondamentale al primo avvio — Keycloak entra in crash-loop se non riesce a raggiungere PostgreSQL.

I backup sono configurati nativamente tramite i valori `backups.*` della Helm chart `cluster` (cloudnative-pg/charts `cluster` >=0.8.0): impostando `backups.method: plugin` è la chart stessa a creare `ObjectStore` e uno `ScheduledBackup` giornaliero verso S3 tramite il plugin barman-cloud — nessun manifest di backup scritto a mano accanto alla definizione del cluster.

---

## Strategia di Backup

I backup ora girano su **entrambi** i target, verso l'object store descritto sopra — RustFS su k3d, il bucket esterno su k3s.

- **k3d**: sia il `CronJob` di snapshot Raft di OpenBao sia lo `ScheduledBackup` CNPG di Keycloak inviano i dati a bucket sull'istanza RustFS in-cluster (`openbao-backup` e `keycloak-backup` rispettivamente) — utile per esercitare il percorso di restore in locale senza toccare alcun servizio esterno.
- **k3s**: gira solo il backup CNPG di Keycloak (OpenBao non esiste su k3s — vedi Gestione dei Segreti sopra), verso un bucket S3-compatibile esterno. Endpoint e percorso di destinazione vengono risolti dalla sostituzione di variabili `postBuild` di Flux sulla Kustomization `apps`, a partire dal Secret `objectstore-substitutions`, così nessun dettaglio del bucket è hardcoded in Git.

---

## Monitoring: Prometheus, Grafana e Log

La Helm release `kube-prometheus-stack` distribuisce Prometheus, Grafana e Alertmanager (questo solo k3s) nel namespace `monitoring`.

[Prometheus](https://prometheus.io) è configurato per fare scraping di tutti i ServiceMonitor e PodMonitor nel cluster, quindi aggiungere monitoring per una nuova app è semplice come distribuire una risorsa `PodMonitor` o `ServiceMonitor`.

Flux stesso è strumentato: PodMonitor dedicati coprono tutti e sei i controller Flux (`source-controller`, `kustomize-controller`, `helm-controller`, `notification-controller`, `image-reflector-controller`, `image-automation-controller`). Dashboard Grafana personalizzate visualizzano lo stato di riconciliazione di Flux e la salute del cluster Kubernetes.

[Grafana](https://grafana.com/oss/grafana) si autentica tramite Keycloak via OIDC, e le sue credenziali admin vengono recuperate da OpenBao tramite ExternalSecret. Le notifiche di allerta da Alertmanager sono collegate al sistema di notifiche di Flux, così i fallimenti delle Kustomization possono attivare alert.

### Aggregazione dei Log: Loki + Fluent Bit

[Loki](https://grafana.com/oss/loki) gira in modalità SingleBinary — una replica, persistenza su filesystem su un PVC. È volutamente semplice: l'alta disponibilità sul log store non è un requisito homelab, e SingleBinary evita il sovraccarico operativo di componenti separati per lettura/scrittura/backend. I log sono interrogabili in Grafana tramite LogQL attraverso un datasource preconfigurato.

**Fluent Operator** gestisce Fluent Bit attraverso CRD Kubernetes (`FluentBit`, `ClusterFilter`, `ClusterOutput`) invece di una ConfigMap grezza. Le regole di routing dei log sono risorse Kubernetes versionabili; aggiungere un filtro o cambiare il target di output non richiede di toccare la specifica del DaemonSet — Fluent Operator riconcilia automaticamente la configurazione di Fluent Bit. Il DaemonSet legge dal socket dei log di containerd su ogni nodo e invia all'API push di Loki. Entrambi i componenti espongono ServiceMonitor così Prometheus raccoglie anche le loro metriche interne.

---

## Bootstrap: Da Zero a Cluster Funzionante

Il `mise.toml` definisce la sequenza di bootstrap completa come task eseguibili:

```bash
# 1. Genera .env con segreti casuali
mise run env-init <github_user> <github_token>

# 2. Installa la CA mkcert e crea il cluster k3d
mise run k3d-create

# 3. Bootstrap Flux (installa flux-operator, crea il secret GitHub, applica FluxInstance)
mise run flux-bootstrap k3d

# 4. Attendi che openbao-0 sia Ready, poi inizializza OpenBao
mise run bao-init   # stampa ROOT_TOKEN — salvalo

# 5. Attendi che Keycloak sia Ready, poi importa il realm
mise run import-realm

# 6. Configura OIDC su OpenBao
mise run config-oidc <ROOT_TOKEN>
```

Dopo il passo 3, Flux prende il controllo e guida il cluster verso lo stato dichiarato. I passi 4–6 sono gli unici interventi manuali necessari: OpenBao deve essere inizializzato prima che ESO possa leggere i segreti, Keycloak deve essere in esecuzione prima che il realm possa essere importato, e la configurazione OIDC richiede entrambi pronti.

---

## Principi di Design Chiave

Guardando questo setup nel suo insieme, emergono alcuni principi chiari:

1. **Tutto è in Git.** Nessuna configurazione fuori banda. Flux riconcilia continuamente.
2. **I segreti non toccano mai Git.** OpenBao (o il vault esterno, in produzione) è la fonte di verità; ExternalSecret fa da ponte verso Kubernetes.
3. **Un unico identity provider.** Keycloak è l'emittente OIDC per ogni applicazione, incluso l'API server.
4. **Il locale rispecchia la produzione.** k3d ha la stessa topologia di k3s — issuer TLS diverso, dominio diverso, ma struttura GitOps identica.
5. **Automazione sui passi manuali.** I task `mise` codificano la sequenza di bootstrap; l'image automation gestisce i deployment.

---

## Cosa Gira

| Componente | Ruolo |
| --- | --- |
| Flux | Motore GitOps |
| Envoy Gateway | Controller Gateway API, ingress |
| cert-manager | Automazione TLS |
| External Secrets Operator | Sincronizzazione segreti da OpenBao / vault esterno |
| OpenBao | Secrets store, auth OIDC (solo k3d) |
| Keycloak | Identity provider OIDC centralizzato |
| CloudNativePG | Operatore PostgreSQL |
| RustFS | Object store S3-compatibile, login OIDC (solo k3d) |
| Prometheus | Raccolta metriche |
| Grafana | Visualizzazione metriche |
| Alertmanager | Routing alert (solo k3s) |
| Loki | Aggregazione log |
| Fluent Bit | Raccolta e invio log (Fluent Operator) |

---

## Conclusione

Questo homelab non è particolarmente semplice — ma è esattamente questo il punto. Dimostra che un singolo operatore può costruire e mantenere un'infrastruttura che sarebbe a suo agio in un ambiente di produzione professionale: guidato dal GitOps, privo di segreti in Git, autenticato tramite SSO, con TLS automatizzato e osservabile.

La combinazione di **FluxCD** per GitOps, **OpenBao/un vault esterno** per i segreti, **Keycloak** per l'identità e **Envoy Gateway** per il networking rappresenta uno stack coerente e ben integrato in cui ogni componente ha un ruolo chiaro. Il pattern Kustomize multi-ambiente mantiene la configurazione DRY senza sacrificare la flessibilità specifica per ambiente.

Se stai costruendo il tuo homelab e vuoi andare oltre `kubectl apply -f` e segreti gestiti manualmente, questa architettura è un buon punto di riferimento — pensiero da produzione, scala da homelab.
