---
title: "Eseguire n8n su un cluster Kubernetes locale: un setup homelab di livello produzione"
date: 2026-04-09
draft: false
tags: ["n8n", "kubernetes", "homelab", "helm", "helmfile", "k3d", "automation", "llm", "ai"]
description: "Come deployare n8n in un cluster Kubernetes locale con queue mode, Helmfile, Envoy Gateway, CNPG e Valkey — con un servizio di inferenza LLM locale — con due comandi."
ShowToc: true
---

[n8n](https://n8n.io) è un potente strumento open-source per l'automazione dei workflow. Eseguirlo in un vero cluster Kubernetes — anche in locale — costringe ad affrontare gli stessi problemi che si troverebbero in produzione: gestione dei segreti, esecuzione a coda, terminazione TLS, proxying WebSocket e isolamento dei task runner. Questo post illustra un setup homelab completamente dichiarativo che affronta tutto questo con serietà, aggiungendo anche un servizio di inferenza LLM locale in modo che i nodi AI nei workflow non escano mai dalla macchina.

Il codice sorgente completo è un piccolo repository infrastructure-as-code. **Non** è l'applicazione n8n — è la configurazione Helm e Kubernetes che la deploya su un cluster K3D locale all'indirizzo `https://n8n.homelab.localhost`.

---

## Strumenti e prerequisiti

Tutte le versioni degli strumenti sono gestite tramite [mise](https://mise.jdx.dev/), dichiarate in `mise.toml`:

```toml
[tools]
helm      = "3"
helmfile  = "latest"
k3d       = "latest"
kubectl   = "latest"
mkcert    = "latest"
```

Eseguire `mise trust && mise install` una sola volta per installare tutto. Dopodiché il cluster viene avviato con due task:

```bash
mise run env-init    # genera REDIS_PASSWORD e N8N_ENCRYPTION_KEY in .env (una sola volta)
mise run k3d-create  # crea il cluster e deploya tutto
```

`env-init` scrive un file `.env` (gitignored) con credenziali generate casualmente. `k3d-create` installa la CA di mkcert nel trust store di sistema, crea il cluster ed esegue `helmfile sync`. Gli hook di Helmfile si occupano del resto: gli hook presync creano namespace e segreti, quelli postsync applicano le risorse Gateway API — nessun passo manuale aggiuntivo.

---

## Architettura del cluster (`k3d.yaml`)

Il cluster ha 1 nodo server e 3 nodi agent. Il nodo server è taintato:

```yaml
- arg: "--node-taint=CriticalAddonsOnly=true:NoExecute"
  nodeFilters:
    - server:*
```

Questo fa sì che tutti i workload applicativi — n8n, il gateway, PostgreSQL, Valkey — vengano schedulati automaticamente sui nodi agent. Il server esegue solo i componenti built-in di k3s (CoreDNS, local-path-provisioner, metrics-server), che tollerano già quel taint. Questo rispecchia un pattern di produzione in cui i nodi control-plane sono riservati all'infrastruttura.

I volumi di storage sono montati solo sui nodi agent:

```yaml
volumes:
  - volume: /tmp/storage:/var/lib/rancher/k3s/storage
    nodeFilters:
      - agent:*
```

La porta 443 è forwardata tramite il load balancer di k3d, quindi HTTPS funziona direttamente dal browser.

Il cluster avvia anche un proxy locale per il registry Docker (`docker-io:5000`) che fa caching delle immagini da Docker Hub. Su connessioni lente o a consumo questo fa risparmiare molto tempo alla ricreazione del cluster.

Traefik, l'ingress predefinito di k3s, è esplicitamente disabilitato — lo stack usa Envoy Gateway al suo posto.

---

## Release Helm (`helmfile.yaml`)

Sette release vengono deployate in ordine di dipendenza:

| Release | Namespace | Scopo |
| --- | --- | --- |
| `cert-manager` | `cert-manager` | Emissione di certificati TLS con supporto Gateway API |
| `envoy-gateway` | `envoy-gateway-system` | Implementazione Gateway API basata su Envoy (v1.8.1) |
| `cnpg` | `cnpg-system` | Operatore CloudNative PostgreSQL |
| `n8n-cnpg` | `n8n` | Cluster PostgreSQL per n8n |
| `n8n-valkey` | `n8n` | Backend di coda compatibile Redis |
| `n8n` | `n8n` | Applicazione n8n |

La direttiva `needs` di Helmfile impone l'ordinamento. Ogni release attende le proprie dipendenze prima di installarsi. È importante: n8n non deve avviarsi prima che database e coda siano pronti.

Envoy Gateway viene installato dal suo chart OCI (`docker.io/envoyproxy/gateway-helm`) come release singola — a differenza di alcune implementazioni gateway non richiede un chart separato per i CRD. I CRD della Gateway API sono bundled nel chart stesso, quindi non vanno pre-installati separatamente: una versione più recente dei CRD nel cluster attiva una `ValidatingAdmissionPolicy` che blocca l'installazione dei CRD (più vecchi) inclusi nel chart.

Il cluster PostgreSQL viene provisionato tramite il chart Helm dell'operatore CNPG (`cnpg/cluster`) e il blocco `initdb` installa l'estensione `pgvector`, così i workflow legati all'AI possono salvare embedding vettoriali direttamente nello stesso database se necessario.

L'operatore LLM (LLMKube) è intenzionalmente tenuto fuori da `helmfile sync` e deployato separatamente:

```bash
mise run llmkube-deploy
```

Questo installa il chart Helm di LLMKube e applica `model.yaml` in un solo passo. Tenerlo separato fa sì che lo stack n8n principale si avvii in modo pulito e indipendente dal download di un modello da diversi gigabyte.

---

## n8n in queue mode

La queue mode è il modello di esecuzione di produzione per n8n. Invece che il processo principale esegua i workflow direttamente, pod worker dedicati prelevano i job da una coda Bull supportata da Valkey (un fork di Redis). Un deployment separato di webhook-processor gestisce in modo indipendente i trigger HTTP in ingresso, così un picco di traffico webhook non compete mai con il carico dell'editor/API sul pod main.

```yaml
queueMode:
  enabled: true
  workerReplicaCount: 3

webhookProcessor:
  enabled: true
  replicaCount: 3
  disableProductionWebhooksOnMainProcess: true
```

Entrambi i replica count sono impostati a 3, in linea con i tre nodi agent del cluster. Il chart deploya worker e webhook processor come normali `Deployment` con un numero fisso di repliche invece di fissare un pod per nodo — non esiste una modalità DaemonSet — quindi si tratta di una scelta di dimensionamento, non di una topologia imposta. `disableProductionWebhooksOnMainProcess` impedisce al pod main di gestire anche il traffico webhook di produzione una volta che i pod processor dedicati se ne fanno carico.

### Task runner esterni

Il modello di task runner di n8n isola l'esecuzione del codice (JavaScript, Python) in processi separati. Il chart esegue un sidecar `task-runner` accanto ai container main e worker — non al webhook-processor, che non esegue mai nodi Code:

```yaml
taskRunners:
  enabled: true
  nativePythonRunner: true
  image:
    repository: n8nio/runners
    tag: "2.27.3-distroless"
```

`nativePythonRunner: true` è il valore predefinito e imposta automaticamente `N8N_NATIVE_PYTHON_RUNNER` su ogni pod che riceve il sidecar — nessun post-renderer o configurazione custom del launcher necessaria per nessuno dei due linguaggi. Il tag dell'immagine distroless è fissato esplicitamente per allinearlo alla versione dell'immagine n8n e mantenere l'immagine rafforzata (senza shell, senza package manager).

Il broker con cui dialoga il sidecar è autenticato: il chart genera un `N8N_RUNNERS_AUTH_TOKEN` casuale alla prima installazione, lo salva in un Secret (`n8n-task-runners`) e lo riutilizza tra un upgrade e l'altro tramite la funzione `lookup` di Helm, così un rolling update della release non invalida i runner in esecuzione.

Le definizioni dei runner per entrambi i linguaggi sono già incluse in `/etc/n8n-task-runners.json` all'interno dell'immagine.

### Binary data in modalità database

n8n supporta diversi backend per lo storage dei dati binari. La configurazione di storage nativa del chart copre solo `filesystem` e `s3` — ma `filesystem` è incompatibile con la queue mode (worker e webhook processor su pod diversi non possono condividere un disco locale). La modalità corretta per questo setup è `database`, che salva i dati binari come BLOB in PostgreSQL.

Non essendoci un campo strutturato dedicato, viene impostata come variabile d'ambiente grezza tramite `config.extraEnv`, applicata in modo identico ai deployment main, worker e webhook-processor:

```yaml
config:
  extraEnv:
    - name: N8N_DEFAULT_BINARY_DATA_MODE
      value: "database"
```

---

## TLS e routing

La catena TLS è:

1. **mkcert** genera una CA locale e la installa nel trust store di sistema e dei browser. Il certificato e la chiave della CA vengono salvati nel Secret `cert-manager/ca-cert` dall'hook presync di `cert-manager`.
2. **cert-manager** usa un `ClusterIssuer` supportato da quel secret per emettere un certificato wildcard per `*.homelab.localhost`, salvato nel Secret `default/gateway-cert`.
3. **Envoy Gateway** termina il TLS sulla porta 443 usando quel certificato.
4. Due regole **HTTPRoute** dividono il traffico verso `n8n.homelab.localhost` tra i due deployment di n8n esposti da un servizio: i path di webhook, form e MCP vanno al servizio webhook-processor, tutto il resto va al servizio main.

Poiché la CA di mkcert viene aggiunta automaticamente al trust store di sistema, il certificato è attendibile da browser e curl senza alcuna configurazione manuale.

Tutte le risorse Gateway API (`GatewayClass`, `Gateway`, `ClusterIssuer`, `HTTPRoute`) vengono applicate inline tramite gli hook postsync di helmfile — non esiste un file manifest separato da mantenere. La `GatewayClass` in particolare non viene creata automaticamente dal chart Helm di Envoy Gateway e deve essere applicata dopo l'installazione del chart:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

### Supporto WebSocket

L'interfaccia di n8n mantiene una connessione WebSocket persistente su `/rest/push` per gli aggiornamenti in tempo reale. A differenza di alcune implementazioni gateway, Envoy Gateway gestisce nativamente gli upgrade WebSocket — non è necessaria alcuna policy aggiuntiva per abilitare il protocollo. La Gateway API standard supporta un campo `timeouts.request` per regola nell'`HTTPRoute`; impostarlo a `0s` disabilita il timeout per-request:

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

Questa suddivisione dei path rispecchia il routing che il chart stesso configurerebbe con un normale `Ingress` Kubernetes, reimplementato come regole Gateway API dato che Envoy Gateway usa `HTTPRoute`. Senza l'override del timeout a `0s`, Envoy chiude il WebSocket allo scadere del timeout idle predefinito e l'interfaccia inizia a mostrare errori di connessione. Questo sostituisce la CRD Envoy-specifica `BackendTrafficPolicy` che era precedentemente necessaria per ottenere lo stesso effetto.

---

## Gestione dei segreti

Le credenziali vengono generate localmente e non vengono mai committate nel repository. Prima di creare il cluster, eseguire:

```bash
mise run env-init
```

Questo scrive un file `.env` (gitignored) con valori generati casualmente:

```
REDIS_PASSWORD=<openssl rand -hex 16>
N8N_ENCRYPTION_KEY=<openssl rand -hex 32>
```

mise carica questo file automaticamente tramite `_.file = ".env"` in `mise.toml`. Gli hook presync di helmfile creano poi i secret Kubernetes a partire da questi valori — nessun comando `kubectl` manuale è necessario.

Cinque segreti vengono utilizzati a runtime:

- **`ca-cert`** (`cert-manager`): certificato e chiave della CA mkcert. cert-manager li legge per firmare il certificato TLS wildcard.
- **`valkey-auth`**: contiene `redis-password`. Usato sia dal chart Valkey (autenticazione ACL) che dal chart n8n (`redis.passwordSecret`).
- **`n8n-core-secrets`**: contiene `N8N_ENCRYPTION_KEY`, `N8N_HOST`, `N8N_PORT` e `N8N_PROTOCOL`. Il chart ha hardcoded il lookup di tutte e quattro le chiavi su questo unico secret (`secretRefs.existingSecret`), quindi devono esistere tutte anche se solo la chiave di cifratura è realmente sensibile. Perdere `N8N_ENCRYPTION_KEY` rende irrecuperabili tutte le credenziali salvate — va trattata come una master password.
- **`n8n-cnpg-cluster-app`**: creato automaticamente dall'operatore CNPG. Contiene la chiave `password` per l'utente database di n8n, consumata direttamente da `database.passwordSecret` — nessuna particolarità da gestire.
- **`n8n-task-runners`**: generato dal chart stesso, non da un hook di helmfile. Contiene il token casuale `N8N_RUNNERS_AUTH_TOKEN` usato dai sidecar task-runner per autenticarsi al broker nei container main/worker.

---

## Hardening della sicurezza

Diverse misure di sicurezza vengono applicate su tutti i tipi di pod, per lo più come variabili d'ambiente grezze sotto `config.extraEnv` dato che il chart non le espone come campi dedicati:

| Misura | Valore |
| --- | --- |
| REST API pubblica | Disabilitata (`N8N_PUBLIC_API_DISABLED: true`) |
| Community packages | Disabilitati |
| Protezione SSRF | Abilitata — blocca le richieste agli indirizzi RFC 1918 / loopback dai nodi workflow |
| Cookie sicuro | Abilitato — HTTPS-only, SameSite-strict |
| Telemetria | Completamente disabilitata (diagnostics, notifiche di versione, template fetching, frontend hooks) |
| Sidecar task runner | Immagine distroless, non-root (UID 1000 tramite il security context del pod), nessuna privilege escalation, tutte le capability Linux rimosse |

La protezione SSRF merita attenzione in un contesto homelab: senza di essa, un nodo workflow potrebbe fare richieste HTTP ad altri servizi della rete locale (NAS, pannello admin del router, altri servizi Kubernetes). Abilitarla blocca questa classe di attacchi anche dai workflow scritti in proprio — un guardrail utile.

---

## Predisposizione al monitoraggio

Il chart non include alcun template `ServiceMonitor`, quindi la raccolta delle metriche non è predisposta di default. Per attivarla bisogna deployare `kube-prometheus-stack` nel namespace `monitoring` e aggiungere un `ServiceMonitor` scritto a mano tramite un hook postsync di helmfile, allo stesso modo in cui oggi viene applicato l'`HTTPRoute` per n8n.

---

## Inferenza LLM locale con LLMKube

Lo stack include [LLMKube](https://github.com/defilantech/LLMKube), un operatore Kubernetes che gestisce il ciclo di vita degli LLM: scarica il file del modello, provisiona il server di inferenza ed espone un servizio ClusterIP — tutto a partire da una coppia di CRD.

### Deploy del modello (`model.yaml`)

Due risorse definiscono il setup di inferenza:

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

La risorsa `Model` avvia un job di download; `InferenceService` poi preleva il GGUF dalla cache e avvia un server llama.cpp. Al primo deploy ci vuole del tempo — il modello pesa diversi gigabyte. Le ricreazioni successive del cluster riutilizzano il PVC in `llmkube-system` (10Gi, provisionato da `mise run llmkube-deploy`).

### Usare l'LLM nei workflow di n8n

Il servizio di inferenza espone un'API compatibile con OpenAI, quindi funziona con il tipo di credenziale OpenAI integrato in n8n. Creare una credenziale di tipo **OpenAI** con:

- **Base URL**: `http://gemma-e2b-service.llmkube-system.svc.cluster.local:8080/v1`
- **API Key**: qualsiasi stringa non vuota (llama.cpp non la valida)

Conviene usare il nome DNS interno di Kubernetes invece del hostname esterno. Dall'interno del cluster, `gemma-e2b-service.llmkube-system.svc.cluster.local` si risolve direttamente al ClusterIP — senza passare per il gateway.

**Attenzione alla protezione SSRF**: `N8N_SSRF_PROTECTION_ENABLED: true` blocca le richieste HTTP verso indirizzi RFC 1918, che include il CIDR dei servizi del cluster. L'URL del servizio diretto sopra sarà bloccato per impostazione predefinita. Per consentirlo, usare `N8N_SSRF_ALLOWED_HOSTNAMES` in `n8n.yaml`:

```yaml
config:
  extraEnv:
    - name: N8N_SSRF_ALLOWED_HOSTNAMES
      value: gemma-e2b-service.llmkube-system.svc.cluster.local
```

L'hostname allowlist ha precedenza sul blocco degli IP, quindi n8n salta completamente il controllo RFC 1918 per questo host. Il DNS interno di Kubernetes è interamente sotto il nostro controllo, che è esattamente il caso d'uso per cui questo tipo di allowlist è pensato.

Con questa configurazione, qualsiasi nodo n8n che accetta una credenziale OpenAI — AI Agent, Chat, Summarize, ecc. — funziona con il modello locale. Nessun traffico esce dalla macchina.

---

## Conclusioni

Questo setup tratta un deployment homelab locale con la stessa serietà di uno di produzione. Queue mode con pod worker e webhook-processor dedicati, segreti cifrati, protezione SSRF, un gateway Envoy appropriato, task runner isolati e autenticati in container distroless — niente di tutto ciò è strettamente necessario per un'installazione locale a singolo utente, ma è tutto buona pratica e costa quasi nulla con un approccio dichiarativo.

Il layer LLMKube aggiunge un servizio di inferenza locale che si innesta nel sistema di credenziali OpenAI già presente in n8n. I workflow che usano nodi AI girano interamente nel cluster: nessuna API key, nessuna chiamata esterna, nessun dato che esce dalla macchina. L'estensione pgvector nel cluster PostgreSQL è già lì se si vuole aggiungere un vector store al setup.
