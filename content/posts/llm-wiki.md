---
title: "A Wiki That Writes Itself: An LLM-Curated Knowledge Base"
date: 2026-05-11
draft: false
tags: ["llm", "knowledge-management", "obsidian", "wiki", "rag", "markdown"]
description: "An alternative to RAG: a persistent markdown wiki, maintained entirely by an LLM, sitting between the user and the raw sources — with schema discipline, an index, and three simple primitives."
ShowToc: true
---

The most common way to use an LLM over a collection of documents is RAG: you upload the files, ask a question, the model retrieves the relevant chunks and generates an answer. It works, but it has a structural flaw: **knowledge doesn't accumulate**. Every question starts from scratch, reassembling the useful fragments by hand. Nothing gets compiled once and for all; nothing sticks.

This project explores a different idea, taken straight from the [llm-wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) manifesto and adapted to a real domain: a **persistent wiki, maintained entirely by the LLM**, sitting between the user and the raw sources. The sources stay immutable in a `raw/` folder; the wiki itself — entity pages, concept pages, source summaries, syntheses — lives in `wiki/` and is written and curated by the LLM. The schema (a plain `AGENTS.md`) holds conventions and workflows together, and co-evolves with the user.

The IDE is Obsidian, open on one side of the screen; the agent is the "programmer" on the other; the wiki is the codebase. The git repo gives you history, branching, and diffs for free.

---

## Three operations, nothing more

The pattern rests on three minimal primitives.

**Ingest.** The user drops a source into `raw/` and asks for it to be processed. The LLM reads it, discusses key takeaways, writes a summary page in `wiki/sources/`, updates `index.md`, propagates the news across the entity and concept pages involved (typically 10–15 files touched in one pass), and appends a line to `log.md`. One source at a time, with the human in the loop. The tedious part — cross-references, summary updates, contradiction flagging — is exactly what makes traditional wikis get abandoned. Here it costs almost nothing.

**Query.** You ask the wiki a question. A dedicated skill runs in a forked subagent, reads `index.md` first to identify the relevant pages, drills into them, and synthesizes an answer with wikilink citations. If the answer is worth keeping — because it compares three or more sources, because it produces reusable reasoning, because it becomes presentable material — it gets archived in `wiki/syntheses/` as a Marp deck. Otherwise it stays in chat and that's it: no duplicates, no pages that age worse than the originals.

**Lint.** Periodic health check: contradictions between pages, claims superseded by newer sources, orphan pages, concepts mentioned but lacking their own page, missing cross-references. The LLM flags, the human decides what to do.

---

## Why this isn't just "Obsidian plus an assistant"

The difference from a generic AI integration on top of Obsidian lies entirely in **schema discipline**. The project's agent file pins down:

- the folder structure (`raw/`, `wiki/entities/`, `wiki/concepts/`, `wiki/sources/`, `wiki/syntheses/`);
- the content language and filename convention;
- the standard sections of each page type (what goes in an entity, what goes in a source, what goes in a synthesis);
- the rules for when to archive an answer and when not to (the "smell test": *would I ever re-read this page?*);
- the synthesis format (Marp, short slides, wikilinks only on the final "Sources" slide);
- how to cite a raw source and how to link a wiki page.

Without these rules, the LLM writes a different wiki in every session. With these rules, the wiki has a stable shape and every ingest reinforces it instead of eroding it.

---

## Index, log, and the order of the day

Two files form the operational skeleton. **`index.md`** is the content-oriented catalog: one line per page, link and summary, organized by category. It's what the LLM reads first before answering a query — instead of classic embedding-based retrieval — and at this scale it works perfectly well. **`log.md`** is the append-only chronological log: every operation (`ingest`, `query`, `lint`) leaves a row with a parseable header, so `grep "^## \[" log.md | tail -5` instantly shows the most recent activity.

When the wiki outgrows the threshold where the index is enough, there's an optional escape hatch: [qmd](https://github.com/tobi/qmd), a local hybrid search engine (BM25 + vector + LLM rerank) over markdown collections. It lives outside the repo, is purely user-local, and is activated *lazily*: the LLM tries it with `npx --no-install qmd`, and if it fails, it silently falls back on the index without bothering anyone. It's an example of the general principle: optional tools stay outside the repo's contract.

The same logic applies on the query side. When the wiki does not cover a question and the [context7](https://github.com/upstash/context7) plugin is present in the Claude Code session, the query subagent falls back to it automatically — resolves the library, pulls current documentation, and integrates the result into the answer. No configuration, no prompt to the user, no changes to the repo. If the plugin isn't installed, the gap is flagged and an ingest is suggested instead. Optional tooling, same philosophy.

---

## The point that changes everything

The tedious part of a knowledge base isn't reading or thinking — it's **bookkeeping**. Updating cross-references, keeping summaries aligned, noting when new data contradicts old, maintaining consistency across dozens of pages. Humans abandon wikis because the maintenance cost grows faster than the value. LLMs don't get bored, don't forget to update a link, and can touch 15 files in one pass.

The human's job stays intact: curate sources, direct the analysis, ask the right questions, decide what it all means. Everything else — the grunt work that makes the difference between scattered notes and organized knowledge — the machine does.
