---
title: "I miei Dotfiles: un Ambiente di Sviluppo per il DevOps"
date: 2026-03-31
draft: false
tags: ["dotfiles", "devops", "chezmoi", "fish", "neovim", "tooling", "mise"]
description: "Come gestisco i miei dotfiles con chezmoi e la toolchain su cui mi sono assestato dopo anni di iterazioni."
ShowToc: true
---

Gestire un ambiente di sviluppo su più macchine è un problema risolto — se si è disposti a investire un po' di tempo all'inizio. Questo post spiega come gestisco i miei dotfiles usando [chezmoi](https://www.chezmoi.io/) e la toolchain su cui mi sono assestato dopo anni di iterazioni.

---

## Perché chezmoi?

I gestori di dotfiles esistono in molte varianti: repository git bare, GNU Stow, script shell personalizzati. Ho scelto chezmoi per alcuni motivi:

- **Templating**: un singolo `dot_gitconfig.tmpl` può rendere valori diversi per macchina (email di lavoro vs. email personale) usando i template Go.
- **Niente symlink**: chezmoi copia i file nella destinazione, quindi non basta un `rm` per rompere tutto.
- **Supporto nativo ai segreti**: si integra con i password manager, la cifratura age e le variabili d'ambiente.
- **Binario singolo**: niente Ruby, niente Python, nessuna dipendenza di sistema — solo un binario gestito da mise.

La convenzione di denominazione è semplice: `dot_foo` diventa `~/.foo`, `dot_config/bar/` diventa `~/.config/bar/`. I file template terminano in `.tmpl` e vengono renderizzati prima del deployment.

---

## Bootstrap di una nuova macchina

Tutto parte da un unico script:

```bash
bash debian-startup.sh
```

Questo script si occupa di:

- Installare i pacchetti di sistema base (`curl`, `git`, `ripgrep`, `fd-find`, ecc.)
- Configurare Docker
- Installare Nerd Font (FiraCode)
- Installare mise (il gestore di versioni runtime)
- Clonare e applicare lo starter LazyVim per Neovim
- Impostare Fish come shell predefinita

Da zero a produttivo con un solo comando.

---

## La toolchain

### mise: uno strumento per governarli tutti

Invece di gestire Node con `nvm`, Python con `pyenv`, e così via, uso [mise](https://mise.jdx.dev/). Un singolo `config.toml` fissa le versioni di tutto:

- Runtime: Node 24
- Kubernetes/DevOps: kubectl, helm, k9s, jq, yq
- Editor: neovim (latest)
- Produttività: lazygit, lazydocker, yazi
- Utility: ast-grep, chezmoi, usage

`mise install` e il gioco è fatto. Niente più "funziona sulla mia macchina" per il tooling.

### Fish shell + Tide

Fish offre autosuggestions ed evidenziazione della sintassi out of the box. Faccio largo uso di `abbr` (abbreviazioni) invece degli alias — si espandono inline, così vedi sempre il comando completo prima che venga eseguito.

Alcune preferite:

```fish
abbr -a --set-cursor='%' -- gp 'git add -A && git commit -m "%" && git push'
abbr -a -- dr 'docker run --rm -it'
abbr -a -- kg 'kubectl get -n'
abbr -a --set-cursor='%' -- kr 'kubectl run -it --rm --restart=Never --image=% -- sh'
```

Il flag `--set-cursor` è un trucco specifico di Fish che posiziona il cursore a metà espansione — utile per i messaggi di commit, dove si finisce sempre a scrivere nello stesso punto.

[Tide](https://github.com/ilancosman/tide) fornisce il prompt tramite Fisher — mostra branch e stato git, contesto Kubernetes e durata dei comandi senza rumore di configurazione.

### Neovim (LazyVim)

Uso [LazyVim](https://www.lazyvim.org/) come base e aggiungo un piccolo insieme di plugin:

- **bufferline** — navigazione tra i buffer simile alle tab con Shift+frecce
- **smart-splits** — navigazione tra i pannelli con Alt+frecce
- **nvim-spider** — movimenti `w`/`b`/`e` più intelligenti che rispettano camelCase e underscore
- **snacks** — fuzzy picker configurato per includere i file nascosti di default
- **tokyonight** — tema in modalità trasparente, così si vede lo sfondo del terminale

### k9s

[k9s](https://k9scli.io/) è una TUI per Kubernetes. La mia configurazione aggiunge:

- **Hotkey personalizzate**: Shift+0–9 saltano a nodi, servizi, pod, HelmRelease, Kustomization
- **Alias**: `dp` → deployments, `sec` → secrets, `jo` → jobs, `cr` → clusterroles
- **Plugin di debug**: un tasto avvia un pod di debug sul nodo selezionato con pulizia automatica
- **Skin trasparente**: eredita lo sfondo del terminale

### WezTerm

[WezTerm](https://wezfurlong.org/wezterm/) è il mio emulatore di terminale — accelerato via GPU, cross-platform, configurato in Lua.

Configurazione rilevante:

- FiraCode Nerd Font per ligature e icone
- `Ctrl+Shift+|` / `Ctrl+Shift+?` per split orizzontali/verticali
- `Ctrl+Shift+<` / `Ctrl+Shift+>` per ruotare i pannelli
- `Ctrl+Shift+{` / `Ctrl+Shift+}` per riordinare i tab

### Claude Code

[Claude Code](https://claude.ai/code) è la CLI di Anthropic per Claude ed è una presenza fissa nel mio workflow — occupa il pannello destro della finestra tmux `dev`. Mantengo definizioni di subagenti personalizzati in `dot_claude/agents/`, che vengono deployati in `~/.claude/agents/`. Ogni agente è un file markdown con frontmatter YAML che dichiara nome, modello e strumenti consentiti, seguito da un system prompt mirato:

- **Coder** (Sonnet) — scrive Terraform, manifest Kubernetes, script Bash e tooling Python. Conosce le convenzioni a memoria, così non devo ripeterle.
- **Planner** (Opus) — esplora il codebase e la documentazione, identifica i rischi, produce un piano di implementazione ordinato. Non tocca mai il codice.
- **Security Reviewer** (Opus) — revisiona i diff infrastrutturali per IAM eccessivamente permissivi, segreti esposti, misconfigurazioni RBAC e problemi di sicurezza dei container. Solo segnalazioni, nessuna correzione.

La separazione tra Planner e Coder rispecchia la disciplina "pensa prima, poi agisci" che cerco di applicare manualmente — solo che ora è imposta da system prompt separati e scelte di modello distinte.

### Tmux

Tmux è il layer di sessione. Lo script `devops.sh` costruisce un layout a due finestre:

- **dev**: Neovim a sinistra, Claude a destra
- **ops**: k9s in alto, una shell in basso

Ogni pannello avvia la propria app come processo top-level (non via `send-keys`), con `remain-on-exit on`. Quando l'app termina, il pannello diventa "morto" invece di chiudersi — `prefix + R` lo fa ripartire con il comando originale. Plugin via TPM: resurrect, continuum, yank, vim-tmux-navigator.

---

## Workflow

Il flusso quotidiano è:

```bash
# Modifica una config nella directory sorgente
chezmoi edit config.fish

# Anteprima delle modifiche
chezmoi diff

# Applica alla home directory
chezmoi apply

# Commit e push
chezmoi git -- add -A && chezmoi git -- commit -m "..." && chezmoi git -- push
```

Su una nuova macchina:

```bash
mise x chezmoi -- chezmoi init --apply thewalterman
```

Tutto qui. Ambiente completo in pochi minuti.

---

## Conclusione

Un buon setup di dotfiles è invisibile quando funziona — ti siedi davanti a una nuova macchina e sembra già casa tua. chezmoi gestisce la meccanica, mise fissa il tooling, Fish + Tide mantengono la shell veloce e informativa. L'investimento iniziale è qualche ora; il ritorno è su ogni macchina, per sempre.

Se sei curioso, il setup completo è su GitHub all'indirizzo [thewalterman/dotfiles](https://github.com/thewalterman/dotfiles).
