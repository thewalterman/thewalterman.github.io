---
title: "Blindare l'Homelab: Chiudere Tutte le Porte"
date: 2026-08-10
draft: false
tags: ["cloudflare", "tailscale", "k3s", "docker", "security", "networking", "opentofu", "homelab"]
description: "Chiudere le porte pubbliche rimaste aperte sul nodo dell'homelab — la 443 dietro un Cloudflare Tunnel con un bug di mismatch SNI da inseguire, e la 22 dietro un sidecar Tailscale in Docker con una trappola di userspace networking lungo la strada."
ShowToc: true
---

L'[homelab](/posts/homelab/) e la sua [infrastruttura OCI](/posts/oci-tofu/) girano con due porte aperte verso internet: la 443, proxata da Cloudflare ma comunque raggiungibile direttamente sul nodo dietro di essa, e la 22, SSH in chiaro sull'IP pubblico. Entrambe restavano superficie d'attacco che non aveva motivo di esistere. Questo è il racconto della chiusura di entrambe — una porta alla volta, con una deviazione di debugging su ciascuna.

---

## Parte 1: Cloudflare Tunnel — Chiudere la 443

### Due chart per il tunnel, filosofie opposte

Il repo ufficiale [`cloudflare/helm-charts`](https://github.com/cloudflare/helm-charts) offre due chart per lo stesso scopo, costruiti su modelli opposti:

| | `cloudflare-tunnel` (locally-managed) | `cloudflare-tunnel-remote` |
| --- | --- | --- |
| Config routing | `values.yaml` — ConfigMap in cluster, ingress rules in git | Dashboard/API Cloudflare, fuori dal cluster |
| Risorse k8s | Deployment + Secret + ConfigMap | Deployment + Secret (solo token) |
| Credenziali richieste | account, tunnel name, tunnel ID, secret | un singolo tunnel token |

**Decisione: `cloudflare-tunnel-remote`.** Meno di 50 righe di template in tutto, l'immagine `cloudflared` resta su `latest` di default quindi non invecchia mai, e mantiene quasi nessuna configurazione Cloudflare-specifica versionata nel repo. Il compromesso: le ingress rules vivono nel dashboard/API di Cloudflare invece che in git. Per un'unica route wildcard verso il Gateway del cluster, è valsa la pena.

**Un effetto collaterale da segnalare**: cloudflared ha bisogno di un nome stabile per raggiungere il Service di Envoy Gateway dentro il cluster, ma Envoy Gateway genera di default un nome di Service non deterministico, con suffisso hash. Risolto con una risorsa `EnvoyProxy`, che fissa il nome del Service a `envoy-gateway`.

### Bad Gateway: un mismatch di SNI

Instradare `*.example.com` verso `https://envoy-gateway.envoy-gateway-system.svc.cluster.local:443` produceva un secco **502 Bad Gateway** su ogni hostname:

```
error="Unable to reach the origin service. [...]: read tcp [...]: connection reset by peer"
originService=http://envoy-gateway.envoy-gateway-system.svc.cluster.local:443
```

La causa: Envoy Gateway sceglie il filter chain in base all'SNI del client, e matchano solo nomi sotto `*.example.com`. Senza un `origin_server_name` esplicito, il client TLS di cloudflared invia come SNI l'*hostname interno del Service* (`envoy-gateway.envoy-gateway-system.svc.cluster.local`) — che non matcha nessun filter chain, quindi Envoy chiude la connessione ancora prima che l'handshake finisca.

Il routing verso il backend giusto dentro Envoy dipende dall'header `Host` della richiesta, non dall'SNI usato per l'handshake — quindi un qualsiasi hostname statico sotto `*.example.com` funziona come `origin_server_name`, che esista o no come record DNS o route reale (qui `tunnel.example.com`). Deve solo sopravvivere all'handshake TLS; il routing per-app continua a funzionare via `Host`.

### Applicare il fix con OpenTofu

La configurazione di ingress del tunnel, incluso `origin_server_name`, è gestita interamente via OpenTofu — lo stesso strumento che gestisce già il resto di questa zona Cloudflare (DNS, WAF), con `cloudflare_zero_trust_tunnel_cloudflared_config` come risorsa dedicata.

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

La regola finale senza `hostname` è un fallback obbligatorio — il provider rifiuta la config senza. Dopo `tofu apply`, `originServerName` compariva nei log di cloudflared, e `curl` su `monitoring.example.com` smetteva di resettare la connessione e otteneva una risposta reale.

Il vecchio record **A** del wildcard — un residuo da prima del tunnel, che puntava direttamente all'IP pubblico — è stato sostituito con un **CNAME** che punta al tunnel:

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

`tofu plan` finale: pulito.

Con il traffico confermato attraverso il tunnel, la 443 è stata rimossa dalla security list di default.

---

## Parte 2: SSH via Tailscale — Chiudere la 22

L'altra porta aperta era la 22. Tailscale era già installato sul nodo, tramite lo script ufficiale, e unito al tailnet come mesh VPN. L'obiettivo era raggiungerlo via Tailscale tramite Docker e poi chiudere del tutto la 22 verso internet.

### Un sidecar Tailscale in Docker

Pattern standard dalla [doc ufficiale di Tailscale](https://tailscale.com/docs/features/containers/docker/how-to/connect-docker-container): un container con l'immagine `tailscale/tailscale` si unisce al tailnet e funge da sidecar di rete per altri container.

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

### Un container usa-e-getta che condivide la rete del sidecar

Serve un container con un client SSH che condivida il network namespace del sidecar, lanciato contro l'IP Tailscale del nodo:

```bash
docker run --rm -it --network container:tailscale kroniak/ssh-client ssh ubuntu@<ip-tailscale>
```

`--network container:tailscale` condivide interfacce e rotte con il sidecar — stessa idea del network namespace condiviso tra container in un pod Kubernetes.

### Il bug: userspace networking

La connessione TCP andava in timeout puro — niente "connection refused," niente richiesta di password, silenzio totale. Diagnosi:

```bash
docker exec tailscale ps aux | grep tailscaled
# tailscaled --tun=userspace-networking   <-- qui
```

Senza un device TUN passato al container, `tailscaled` cade di default in **userspace networking**: nessuna vera interfaccia `tailscale0` nel network namespace. I comandi propri di Tailscale (`tailscale ping`) continuano a funzionare — passano dallo stack interno di `tailscaled` — ed è esattamente questo che rende la trappola insidiosa: lo strumento che normalmente proverebbe la connettività continua a funzionare proprio mentre il problema reale è altrove. Solo `ps aux` e `ip a` dentro il sidecar lo rivelano. I socket kernel ordinari — `ssh`, `nc`, qualsiasi cosa da un container che condivide la rete — non hanno alcuna via d'uscita, da qui il timeout silenzioso.

Fix, e servono entrambe le parti insieme:

```yaml
services:
  tailscale:
    # ...
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - TS_USERSPACE=false   # senza questo, il device tun viene passato ma ignorato
```

Con l'accesso via Tailscale verificato, la regola di ingresso SSH pubblica è stata rimossa dalla security list di default.

---

## Cosa è davvero chiuso adesso

| Porta | Prima | Ora |
| --- | --- | --- |
| 443 | Aperta sul nodo, con Cloudflare a fare proxy davanti | Chiusa — il traffico raggiunge il nodo solo tramite il Cloudflare Tunnel |
| 22 | Aperta verso `0.0.0.0/0` | Chiusa — accesso solo via Tailscale, decapsulato localmente su UDP già permesso |

## Takeaway

- **Cloudflare Tunnel**: il chart "remote" vince per un homelab proprio *perché* mantiene quasi nulla di Cloudflare-specifico in git — il compromesso è che le ingress rules vivono nell'API di Cloudflare invece che in una ConfigMap, motivo per cui gestirle via OpenTofu fin dal primo giorno conta.
- **SNI vs `Host`**: la scelta del filter chain di Envoy si basa sull'SNI TLS, mentre il routing effettivo si basa sull'header HTTP `Host` — due campi diversi con due ruoli diversi, facili da confondere quando solo uno dei due sta fallendo.
- **Tailscale in Docker**: l'userspace networking è la trappola che si nasconde dietro il fatto che gli strumenti diagnostici di Tailscale stessi continuano a funzionare mentre tutto il resto va in timeout silenzioso.
- Entrambi i fix condividono la stessa forma: il traffico incapsulato/tunnelato (la connessione uscente del Cloudflare Tunnel, l'UDP di Tailscale) non tocca mai la security list o la regola firewall che viene cancellata — che è esattamente il motivo per cui chiudere la porta è sicuro, non un'assunzione da lasciare non verificata.
