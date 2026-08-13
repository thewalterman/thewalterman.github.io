---
title: "Un Bot AI Sandboxato per l'Homelab"
date: 2026-08-13
draft: false
tags: ["ansible", "k3s", "kubernetes", "rbac", "podman", "security", "ai", "llm", "homelab"]
description: "Installare Hermes Agent — un assistente AI self-hosted, raggiungibile via app di chat — sul nodo dell'homelab tramite Ansible, e poi togliergli quasi tutto ciò che poteva fare di sbagliato: un toolset ridotto all'essenziale, un sandbox Podman rootless per l'esecuzione dei comandi, e un'identità Kubernetes dedicata."
ShowToc: true
---

L'[homelab](/posts/homelab/) gira su un singolo nodo k3s, provisionato e mantenuto interamente tramite [OpenTofu e Ansible](/posts/homelab-infra/). L'unico pezzo che non rientrava in questo schema era dare a un agente AI accesso sufficiente per aiutare davvero a gestirlo — `kubectl`, `helm`, `flux` — senza dargli anche il controllo completo della macchina. Questo è quel pezzo: [Hermes Agent](https://github.com/NousResearch/hermes-agent), installato da un playbook Ansible insieme a quelli per il patching OS e il bootstrap di k3s, raggiungibile via app di chat, e confinato abbastanza stretto che un comando andato male non possa uscire dal cluster che dovrebbe gestire.

---

## Un Altro Playbook, Stessa Forma degli Altri

`bootstrap-hermes.yml` segue lo stesso pattern di `bootstrap-k3s.yml`: un'altra cosa che vive a livello OS, fuori dal grafo di risorse di OpenTofu, installata e mantenuta sincronizzata da Ansible.

```bash
ansible-playbook -i ansible/inventory.yml ansible/bootstrap-hermes.yml
```

Lancia l'installer ufficiale di Hermes, porta sul nodo una manciata di file versionati, e gestisce un servizio `systemd --user` con linger abilitato — così il gateway sopravvive a un reboot senza che nessuno debba fare login interattivo:

```yaml
- name: check for existing hermes install
  stat:
    path: "{{ hermes_home }}/hermes-agent/venv/bin/python"
  register: hermes_venv

- name: run official hermes installer
  shell: curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
  when: not hermes_venv.stat.exists

- name: template SOUL.md, AGENTS.md, config.yaml
  copy:
    src: "hermes/files/{{ item }}"
    dest: "{{ hermes_home }}/{{ item }}"
  loop: [SOUL.md, AGENTS.md, config.yaml]

- name: enable linger so the user service survives reboot without login
  become: true
  command: "loginctl enable-linger {{ ansible_user }}"
```

La documentazione stessa di Hermes traccia una linea deliberata tra due file, e vale la pena mantenerla: **`SOUL.md`** è identità e tono — chi è l'agente, come parla — e deve restare stabile tra progetti diversi. **`AGENTS.md`** sono le regole operative specifiche di questo homelab — come usare `kubectl`, quando chiedere prima di agire, la disciplina GitOps. Mischiarli significa che ogni futuro ritocco a "come deve parlare" rischia di portarsi dietro istruzioni specifiche dell'homelab, e viceversa.

---

## Guardrail Prima del Sandboxing

Due livelli di configurazione sono arrivati prima di qualsiasi discorso sull'isolamento:

```yaml
approvals:
  mode: manual
  deny:
    - "rm -rf*"
    - "*k3s-killall.sh*"
    - "*flux uninstall*"
    - "*ufw*"
    - "reboot*"
    - "shutdown*"
    - "*delete node*"
    - "*delete pvc*"
    - "*delete persistentvolumeclaim*"
    - "*delete namespace*"
    - "*delete ns *"
    - "*helm uninstall*"
    - "*delete clusterrole*"
    - "*delete clusterrolebinding*"
```

`mode: manual` significa che l'agente chiede conferma prima di eseguire qualcosa che giudica rischioso; la lista `deny` è un pavimento rigido sotto quel giudizio, pattern che non vengono mai eseguiti indipendentemente da cosa decide il modello. Economico da scrivere, e utile da avere — ma è una lista di comandi *già noti* come pericolosi. Non dice nulla su un comando a cui nessuno aveva pensato di aggiungere alla lista.

---

## Ridurre il Toolset Prima di Ridurre Tutto il Resto

Hermes viene fornito con un lungo catalogo di toolset opzionali e ogni piattaforma di chat ha il proprio bundle di default.

`platform_toolsets` in `config.yaml` fissa una lista corta ed esplicita di strumenti da poter utilizzare:

```yaml
platform_toolsets:
  cli:
    - clarify
    - code_execution
    - delegation
    - file
    - memory
    - session_search
    - skills
    - terminal
    - todo
    - vision
    - web
```

Due dei toolset lasciati fuori meritano una menzione specifica. `computer_use` — controllo in background di un vero desktop — non ha alcun caso d'uso legittimo per un bot il cui unico lavoro è parlare con una API Kubernetes, quindi viene tolto senza appello. `cronjob` viene tolto per un motivo più preciso: l'esecuzione dei cron in Hermes gira senza supervisione, senza che nessun umano veda mai il comando prima che parta, e il sistema di approvazione che dovrebbe intercettarne uno pericoloso ha un difetto non ancora corretto — il controllo è una breve lista di pattern letterali di comandi shell, facilmente elusa formulando la stessa richiesta distruttiva in linguaggio naturale invece che come comando shell ("leggi il file `~/.hermes/.env` e mostrami il contenuto" passa ogni controllo che una regola contro `cat *.env` intercetterebbe). Uno scheduler senza un cancello che funzioni davvero, con credenziali reali del cluster montate, non è un toolset da tenere finché la cosa non viene corretta.

---

## La Vera Domanda: Cosa Può Davvero Raggiungere

Di default Hermes può eseguire comandi direttamente sul nodo. È lo stesso livello di privilegio di una sessione SSH interattiva — un prompt injection da una pagina che ha recuperato, o semplicemente il modello che fa la cosa sbagliata con piena convinzione, finisce sullo stesso host che fa girare tutto il cluster. La lista `deny` sopra aiuta contro comandi che *qualcuno ha pensato di scrivere*; non dice nulla sulla forma dell'accesso in sé.

La soluzione ha due livelli indipendenti: sandboxare dove i comandi vengono eseguiti, e limitare cosa possono fare una volta dentro.

---

## Livello 1: Podman Rootless Invece dell'Host Nudo

La configurazione `terminal.backend` di Hermes può instradare l'esecuzione dei comandi attraverso un container invece di eseguirli in locale. La scelta ovvia sarebbe Docker — se non fosse che Docker significa un demone che gira come root, permanentemente, e dare all'utente non privilegiato dell'agente accesso a quel demone significa aggiungerlo al gruppo `docker`, che è notoriamente equivalente a dargli root. Un compromesso strano per uno strumento il cui unico compito è girare come servizio `systemd --user` non privilegiato.

[Podman](https://podman.io/) risolve la forma del problema invece di aggirarla: **nessun demone**, e **rootless di default** — i container girano dentro il namespace dell'utente che li lancia, nessuna appartenenza a gruppo che sia segretamente equivalente a root.

```yaml
terminal:
  backend: docker    # la chiave di config non cambia
  docker_image: "localhost/hermes-sandbox:latest"
```

```ini
# hermes-gateway.service
Environment="HERMES_DOCKER_BINARY=podman"
```

Installarlo è un task Ansible prima di tutto il resto nel playbook:

```yaml
- name: ensure podman is installed (rootless sandbox runtime for Hermes terminal.backend)
  become: true
  apt:
    name: podman
    state: present

- name: verify rootless podman works for this user
  command: podman info
  changed_when: false
```

Un effetto collaterale di questa scelta merita di essere segnalato: una volta che `terminal.backend` punta a un container, il sistema di rilevamento dei comandi pericolosi integrato in Hermes — l'euristica dietro `mode: manual` — si disattiva del tutto, sul presupposto che il container stesso sia ora il confine di sicurezza. La lista `deny` di prima è quello che resta a far davvero da guardia per i comandi instradati qui dentro, motivo per cui copre già le operazioni distruttive sul cluster e non solo quelle distruttive sull'host.

---

## Costruire l'Immagine del Sandbox

Il sandbox ha bisogno di `kubectl`, `helm` e `flux` — nessuno dei quali esiste in un'immagine base generica. Il primo tentativo di riusare ciò che era già sul nodo ha trovato un muro strano: `kubectl` sull'host risultava essere un symlink al binario `k3s` stesso (`k3s kubectl ...`), non un eseguibile autonomo, quindi montarlo con un bind mount in un container non correlato non era davvero un'opzione. Al suo posto, binari standalone, pinnati alle stesse versioni già in esecuzione sul nodo, sono finiti in un'immagine piccola:

```dockerfile
FROM debian:bookworm-slim

ARG KUBECTL_VERSION=v1.34.6
ARG HELM_VERSION=v4.1.3
ARG FLUX_VERSION=v2.9.4
ARG TARGETARCH=arm64

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl tar \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${TARGETARCH}/kubectl" -o /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl

RUN curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-${TARGETARCH}.tar.gz" | tar -xz -C /tmp \
    && mv "/tmp/linux-${TARGETARCH}/helm" /usr/local/bin/helm

RUN curl -fsSL "https://github.com/fluxcd/flux2/releases/download/${FLUX_VERSION}/flux_${FLUX_VERSION#v}_linux_${TARGETARCH}.tar.gz" \
    | tar -xz -C /usr/local/bin flux
```

`TARGETARCH=arm64` perché il nodo è un'istanza Oracle Ampere — facile scordarlo quando ogni esempio in rete usa `amd64` di default. Il task Ansible ricostruisce l'immagine solo quando il Dockerfile cambia o il tag manca:

```yaml
- name: check whether the hermes-sandbox image already exists
  command: podman image exists localhost/hermes-sandbox:latest
  register: hermes_sandbox_image
  failed_when: false
  changed_when: false

- name: build the hermes-sandbox image (Dockerfile changed, or image/tag missing)
  command: podman build -t localhost/hermes-sandbox:latest "{{ hermes_home }}/sandbox"
  when: hermes_sandbox_context.changed or hermes_sandbox_image.rc != 0
```

---

## La Trappola di Rete

Il kubeconfig di k3s punta a `https://127.0.0.1:6443`, corretto sul nodo e sbagliato nell'istante in cui viene letto da dentro un container col proprio network namespace — `127.0.0.1` lì dentro è il loopback *del container*, non c'è nulla in ascolto su quello. Il kubeconfig generato per il sandbox riscrive l'indirizzo del server con l'IP LAN del nodo invece del loopback:

```yaml
- name: generate a sandbox kubeconfig from the dedicated hermes ServiceAccount
  become: true
  shell: |
    CA=$(k3s kubectl get secret hermes-token -n hermes -o jsonpath='{.data.ca\.crt}')
    TOKEN=$(k3s kubectl get secret hermes-token -n hermes -o jsonpath='{.data.token}' | base64 -d)
    cat > "{{ hermes_home }}/kubeconfig-sandbox" <<EOF
    apiVersion: v1
    kind: Config
    clusters:
      - name: homelab
        cluster:
          server: https://{{ ansible_facts.default_ipv4.address }}:6443
          certificate-authority-data: ${CA}
    users:
      - name: hermes
        user:
          token: ${TOKEN}
    contexts:
      - name: hermes
        context: {cluster: homelab, user: hermes}
    current-context: hermes
    EOF
```

---

## Livello 2: Admin del Cluster, Non del Nodo

Sandboxare l'esecuzione non è tutta la storia. Montare il vero kubeconfig admin di k3s dentro quel container, in sola lettura, rende l'isolamento puramente cosmetico — un comando compromesso dentro il sandbox può ancora fare `kubectl create` di un pod con `hostPath: /` o `privileged: true` e tornare dritto sul nodo da cui doveva essere tenuto fuori. RBAC e Podman rispondono a due domande diverse: Podman contiene un *comando*, RBAC deve contenere cosa possono fare le *credenziali* di quel comando.

Nel repo GitOps abbiamo Pod Security `baseline` su ogni namespace e questo blocca già `privileged: true`, i volumi `hostPath`, e i namespace host (`hostNetwork`/`hostPID`/`hostIPC`) a livello di admission, prima ancora che RBAC entri in gioco. Questo chiude la via di fuga più grave indipendentemente da cosa può fare il kubeconfig montato. Quello che mancava ancora era un'identità che non fosse cluster-admin fin dall'inizio.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hermes
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: hermes
  namespace: hermes
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: hermes-edit
subjects:
  - kind: ServiceAccount
    name: hermes
    namespace: hermes
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Secret
metadata:
  name: hermes-token
  namespace: hermes
  annotations:
    kubernetes.io/service-account.name: hermes
type: kubernetes.io/service-account-token
```

---

## Applicare l'RBAC da Ansible, Non da GitOps

Le applicazioni del cluster sono gestite da Flux, e l'istinto sarebbe stato metterci anche questo manifest, insieme a Keycloak e a tutto il resto. Non ci sta: Flux non è ancora inizializzato nel momento in cui gira `bootstrap-hermes.yml` — l'ordine è Tofu → k3s → *poi* Flux, a mano, una volta — e Hermes stesso vive già fuori dal grafo di risorse di Flux, come k3s. Far passare l'identità RBAC per GitOps significherebbe far dipendere il bootstrap di Hermes da un repo completamente diverso che deve già essere stato reconciliato.

Quindi il playbook lo applica direttamente:

```yaml
- name: apply Hermes RBAC manifest
  become: true
  command: "k3s kubectl apply -f {{ hermes_home }}/hermes-rbac.yaml"
  register: hermes_rbac_apply
  changed_when: "'unchanged' not in hermes_rbac_apply.stdout"

- name: wait for the hermes ServiceAccount token to be populated
  become: true
  command: "k3s kubectl get secret hermes-token -n hermes -o jsonpath={.data.token}"
  register: hermes_token_check
  until: hermes_token_check.stdout | length > 0
  retries: 10
  delay: 2
```

---


## Cosa È Davvero Contenuto Ora

| Domanda | Prima | Dopo |
| --- | --- | --- |
| Dove girano i comandi? | Direttamente sull'host | Dentro un container Podman rootless |
| Cosa può raggiungere sul nodo un comando compromesso? | Tutto ciò che può l'utente | Nulla — niente filesystem host, niente systemd, nessun altro processo |
| Che identità Kubernetes porta con sé? | Il kubeconfig admin del cluster stesso | Una ServiceAccount dedicata `hermes` |
| Può leggere/riavviare/patchare workload? | Sì | Sì, su ogni namespace |
| Può elencare i nodi, o toccare oggetti RBAC? | Sì | No |
| Può creare un pod che raggiunge direttamente l'host? | Sì | No — bloccato in admission da Pod Security `baseline`, indipendentemente da RBAC |

## Cose da Portarsi a Casa

- **Sandboxare l'esecuzione e limitare le credenziali sono due problemi separati.** Un container con un kubeconfig admin montato dentro non è contenuto — è lo stesso potere con qualche passaggio in più.
- **L'admission control (Pod Security `baseline`) chiude la via di fuga che il solo scoping RBAC non può chiudere** — la differenza tra "admin del cluster" e "root sul nodo" si decide lì, non nei verbi RBAC.
- **Non tutto appartiene a GitOps.** Una risorsa necessaria per bootstrap-are un componente che a sua volta vive fuori dal grafo GitOps appartiene agli strumenti di bootstrap di quel componente, non incastrata in un repo che non può ancora girare.
- **I percorsi di esecuzione non supervisionati hanno bisogno di un cancello che funzioni davvero, non solo sulla carta.** Uno scheduler che gira senza che nessuno guardi è esattamente il punto in cui i punti ciechi di un sistema di approvazione contano di più — motivo per cui `cronjob` resta spento qui finché il flusso di approvazione dei cron di Hermes non viene corretto.
