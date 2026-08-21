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
- Kubernetes/DevOps: kubectl, helm, k9s, jq, yq, stern
- Editor: neovim (latest)
- Produttività: lazygit, lazydocker, yazi
- Utility: ast-grep, chezmoi, tree-sitter, usage

`mise install` e il gioco è fatto. Niente più "funziona sulla mia macchina" per il tooling.

### Fish shell + pure

Fish offre autosuggestions ed evidenziazione della sintassi out of the box. Faccio largo uso di `abbr` (abbreviazioni) invece degli alias — si espandono inline, così vedi sempre il comando completo prima che venga eseguito.

Alcune preferite:

```fish
abbr -a --set-cursor='%' -- gp 'git add -A && git commit -m "%" && git push'
abbr -a -- dr 'docker run --rm -it'
abbr -a -- kg 'kubectl get -n'
abbr -a --set-cursor='%' -- kr 'kubectl run -it --rm --restart=Never --image=% -- sh'
```

Il flag `--set-cursor` è un trucco specifico di Fish che posiziona il cursore a metà espansione — utile per i messaggi di commit, dove si finisce sempre a scrivere nello stesso punto.

Ho sostituito il prompt di default con [pure](https://github.com/pure-fish/pure) — minimale di default, e ogni comportamento è una variabile universale `pure_*` invece che una config file. Tengo `fish_variables` versionato anche nel repo di dotfiles.

### Neovim (LazyVim)

Uso [LazyVim](https://www.lazyvim.org/) come base e aggiungo un piccolo insieme di plugin:

- **bufferline** — navigazione tra i buffer simile alle tab con Shift+frecce
- **smart-splits** — navigazione tra i pannelli con Alt+frecce
- **nvim-spider** — movimenti `w`/`b`/`e` più intelligenti che rispettano camelCase e underscore
- **snacks** — fuzzy picker configurato per includere i file nascosti di default
- **tokyonight** — tema in modalità trasparente, così si vede lo sfondo del terminale

### WezTerm

[WezTerm](https://wezfurlong.org/wezterm/) è il mio emulatore di terminale — accelerato via GPU, cross-platform, configurato in Lua.

Configurazione rilevante:

- FiraCode Nerd Font per ligature e icone
- `Ctrl+Shift+|` / `Ctrl+Shift+?` per split orizzontali/verticali
- `Ctrl+Shift+<` / `Ctrl+Shift+>` per ruotare i pannelli
- `Ctrl+Shift+{` / `Ctrl+Shift+}` per riordinare i tab

### Claude Code

[Claude Code](https://claude.ai/code) è la CLI di Anthropic per Claude ed è una presenza fissa nel mio workflow — occupa il pannello destro della tab zellij `dev`. Mantengo definizioni di subagenti personalizzati in `dot_claude/agents/`, che vengono deployati in `~/.claude/agents/`. Ogni agente è un file markdown con frontmatter YAML che dichiara nome, modello e strumenti consentiti, seguito da un system prompt mirato:

- **Coder** (Sonnet) — scrive Terraform, manifest Kubernetes, script Bash e tooling Python. Conosce le convenzioni a memoria, così non devo ripeterle.
- **Planner** (Opus) — esplora il codebase e la documentazione, identifica i rischi, produce un piano di implementazione ordinato. Non tocca mai il codice.
- **Security Reviewer** (Opus) — revisiona i diff infrastrutturali per IAM eccessivamente permissivi, segreti esposti, misconfigurazioni RBAC e problemi di sicurezza dei container. Solo segnalazioni, nessuna correzione.

La separazione tra Planner e Coder rispecchia la disciplina "pensa prima, poi agisci" che cerco di applicare manualmente — solo che ora è imposta da system prompt separati e scelte di modello distinte.

Oltre agli agenti, `dot_claude/skills/` contiene skill che vincolano le modifiche infra a specifiche invocazioni di tool, invece di affidarsi a passaggi di revisione manuale:

- `tf-fmt-validate`, `tf-plan-diff`, `cost-estimate` — format/validate di Terraform, diff del plan e stima infracost
- `k8s-dry-run`, `rbac-diff` — dry run server-side di Kubernetes/Helm e diff RBAC prima/dopo
- `policy-scan`, `secrets-scan` — scansione statica tfsec/kube-score, scansione segreti con gitleaks
- `shellcheck-gate` — shellcheck su ogni script bash che scrivo o modifico
- `provider-docs-lookup` — recupera la documentazione della versione pinnata per una resource di un provider Terraform, un oggetto API Kubernetes o un chart Helm

Vengono deployate in `~/.claude/skills/` allo stesso modo degli agenti. Ognuna richiede il proprio tool sottostante (`terraform`, `infracost`, `tfsec`, `kube-score`, `gitleaks`, `shellcheck`) su `PATH` tramite mise.

### Zellij

Zellij è il layer di sessione. Ho configurato la modalità prefisso `Ctrl+b` (per la muscle memory di tmux, già integrata in Zellij) e uno scrollback più ampio. Lo script `devops.sh` si aggancia a (o crea) una sessione con nome usando il layout `devops`, un setup a due tab:

- **dev**: Neovim a sinistra, Claude a destra
- **ops**: k9s in alto, una shell in basso

Ogni pannello avvia la propria app direttamente come `command` pane — nessuna configurazione di `remain-on-exit` necessaria, Zellij mantiene già il pannello aperto quando il comando termina e lo fa ripartire con `ENTER`. Una funzione fish `devops` avvolge `zellij attach --create <session> --default-layout devops`, usando come nome della sessione la directory corrente quando `$KUBECONFIG` è impostato.

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

Un buon setup di dotfiles è invisibile quando funziona — ti siedi davanti a una nuova macchina e sembra già casa tua. chezmoi gestisce la meccanica, mise fissa il tooling, Fish + pure mantengono la shell veloce e informativa. L'investimento iniziale è qualche ora; il ritorno è su ogni macchina, per sempre.

Se sei curioso, il setup completo è su GitHub all'indirizzo [thewalterman/dotfiles](https://github.com/thewalterman/dotfiles).
