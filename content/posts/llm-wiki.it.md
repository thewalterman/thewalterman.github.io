---
title: "Un Wiki Che Si Scrive Da Solo: Una Knowledge Base Curata Da un LLM"
date: 2026-05-11
draft: false
tags: ["llm", "knowledge-management", "obsidian", "wiki", "rag", "markdown"]
description: "Un'alternativa al RAG: un wiki markdown persistente, mantenuto interamente da un LLM, che si interpone fra l'utente e le fonti grezze — con disciplina dello schema, un index e tre primitive semplici."
ShowToc: true
---

Il modo più diffuso di usare un LLM su una collezione di documenti è il RAG: carichi i file, fai una domanda, il modello recupera i chunk rilevanti e genera una risposta. Funziona, ma ha un difetto strutturale: **la conoscenza non si accumula**. Ogni domanda riparte da zero, ricomponendo a mano i frammenti utili. Niente viene compilato una volta per tutte; niente resta.

Questo progetto sperimenta un'idea diversa, presa di peso dal manifesto [llm-wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) e adattata a un dominio reale: un **wiki persistente, mantenuto interamente dall'LLM**, che si interpone fra l'utente e le fonti grezze. Le fonti restano immutabili in una cartella `raw/`; il wiki — pagine entità, pagine concetto, riassunti delle fonti, sintesi — vive in `wiki/` ed è scritto e curato dall'LLM. Lo schema (un classico `AGENTS.md`) tiene insieme convenzioni e workflow, e co-evolve con l'utente.

L'IDE è Obsidian, aperto su un lato dello schermo; l'agente è il "programmatore" sull'altro; il wiki è il codebase. Il git repo dà gratis history, branching e diff.

---

## Tre operazioni, niente di più

Il pattern si regge su tre primitive minime.

**Ingest.** L'utente droppa una fonte in `raw/` e chiede di processarla. L'LLM legge, discute i punti chiave, scrive una pagina di riassunto in `wiki/sources/`, aggiorna `index.md`, propaga le novità sulle pagine entità e concetto coinvolte (tipicamente 10–15 file toccati in un colpo solo), e appende una riga in `log.md`. Una fonte alla volta, con l'umano nel loop. La parte tediosa — cross-reference, aggiornamento dei riassunti, segnalazione delle contraddizioni — è esattamente quella che fa abbandonare i wiki tradizionali. Qui costa quasi zero.

**Query.** Si fa una domanda al wiki. Una skill dedicata gira in un subagent forkato, legge prima `index.md` per individuare le pagine rilevanti, vi entra dentro e sintetizza una risposta con citazioni in forma di wikilink. Se la risposta vale la pena di essere conservata — perché confronta tre o più fonti, perché produce un ragionamento riutilizzabile, perché diventa materiale presentabile — viene archiviata in `wiki/syntheses/` come deck Marp. Altrimenti resta in chat e basta: niente duplicati, niente pagine che invecchiano peggio degli originali.

**Lint.** Health-check periodico: contraddizioni fra pagine, claim superati da fonti più recenti, pagine orfane, concetti citati ma senza pagina propria, cross-reference mancanti. L'LLM segnala, l'umano decide cosa fare.

---

## Perché non è solo "Obsidian + un assistente"

La differenza con un'integrazione AI generica su Obsidian sta tutta nella **disciplina dello schema**. Il file agent di progetto fissa:

- la struttura delle cartelle (`raw/`, `wiki/entities/`, `wiki/concepts/`, `wiki/sources/`, `wiki/syntheses/`);
- la lingua dei contenuti e la convenzione dei filename;
- le sezioni standard di ogni tipo di pagina (cosa va in una entity, cosa in una source, cosa in una synthesis);
- le regole su quando archiviare una risposta e quando no (lo "smell test": *rileggerei mai questa pagina?*);
- il formato delle sintesi (Marp, slide brevi, wikilink solo nella slide "Fonti");
- come si cita una fonte grezza e come si linka una pagina del wiki.

Senza queste regole, l'LLM scrive un wiki diverso a ogni sessione. Con queste regole, il wiki ha una forma stabile e ogni ingest la rafforza invece di eroderla.

---

## Indice, log, e l'ordine del giorno

Due file fanno da scheletro operativo. **`index.md`** è il catalogo content-oriented: una riga per pagina, link e summary, organizzato per categoria. È quello che l'LLM legge per primo prima di rispondere a una query — al posto del classico embedding-based retrieval — e a queste dimensioni funziona benissimo. **`log.md`** è l'append-only chronological log: ogni operazione (`ingest`, `query`, `lint`) lascia una riga con header parseable, così `grep "^## \[" log.md | tail -5` dà subito le ultime cose successe.

Quando il wiki cresce oltre la soglia in cui l'index basta, c'è una via di uscita opzionale: [qmd](https://github.com/tobi/qmd), un motore di ricerca locale ibrido (BM25 + vettoriale + rerank LLM) su collezioni markdown. Vive fuori dal repo, è puramente user-local, e si attiva *lazy*: l'LLM ci prova con `npx --no-install qmd`, e se fallisce ricade silenziosamente sull'index senza disturbare. È un esempio del principio generale: gli strumenti opzionali stanno fuori dal contratto del repo.

La stessa logica vale lato query. Se il wiki non copre una domanda e il plugin [context7](https://github.com/upstash/context7) è presente nella sessione Claude Code, il subagent di query vi ricade automaticamente — risolve la libreria, recupera la documentazione aggiornata e integra il risultato nella risposta. Nessuna configurazione, nessun prompt all'utente, nessuna modifica al repo. Se il plugin non è installato, il gap viene segnalato e si suggerisce un ingest. Tooling opzionale, stessa filosofia.

---

## Il punto che cambia tutto

La parte tediosa di un knowledge base non è leggere o pensare — è la **bookkeeping**. Aggiornare cross-reference, tenere allineati i riassunti, segnare quando un dato nuovo contraddice uno vecchio, mantenere consistenza su decine di pagine. Gli umani abbandonano i wiki perché il costo di manutenzione cresce più in fretta del valore. Gli LLM non si annoiano, non si dimenticano di aggiornare un link, e toccano 15 file in un passaggio.

Il lavoro dell'umano resta intatto: curare le fonti, indirizzare l'analisi, fare le domande giuste, decidere cosa significa. Tutto il resto — il grunt work che fa la differenza fra appunti sparsi e una conoscenza organizzata — lo fa la macchina.
