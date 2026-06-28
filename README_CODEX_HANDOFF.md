# Codex Handoff: EvoSemSQL NL2SQL Research Prototype

This folder is designed to be copied into the root of a fresh GitHub repository before handing the work to Codex.

## What to give Codex

1. Copy all Markdown files in this handoff folder into the repository root.
2. Put the Odyssey paper archive at:

```text
research/raw/odyssey.zip
```

3. Export relevant Notion notes as Markdown or HTML and place them under:

```text
research/notion_exports/
```

4. If you have a Deep Research final report, save it as:

```text
docs/deep_research_synthesis.md
```

5. Paste the contents of `CODEX_MASTER_PROMPT.md` into Codex from the repository root.

Do **not** paste a huge unstructured research dump as the main task. Keep the main Codex prompt short and let the Markdown files serve as persistent project context.

## Intended research direction

Working title:

> **EvoSemSQL: Self-Evolving Semantic Memory for Cost-Aware Interactive NL2SQL**

Core idea:

A NL2SQL agent should not treat every user query as a fresh one-shot schema-linking problem. It should build a typed semantic memory graph from database exploration, previous successful SQL traces, business concepts, FAQ-like analytical intents, validated SQL templates, and failure cases. At query time, it should route among:

1. answering from verified memory,
2. instantiating a known SQL template,
3. generating SQL from a compact retrieved semantic subgraph,
4. falling back to a full-schema baseline only when necessary.

The research claim is not merely “use cache” or “use RAG.” The novelty is the combination of typed evolving memory, answerability/staleness gates, graph-constrained context construction, and an evaluation protocol that measures accuracy **and** efficiency under streaming interactive NL2SQL workloads.

## Success criteria for the first Codex pass

A successful first pass should produce:

- A reproducible Python package with tests.
- Paper/PDF ingestion utilities for the Odyssey zip.
- A literature matrix generated from the PDFs.
- A working SQLite-based prototype of the memory graph and router.
- At least one toy NL2SQL fixture proving memory hits, template hits, and fallback generation paths.
- Dataset adapter stubs for Spider 2.0, BIRD, and BIRD-Interact.
- Benchmark runner that logs accuracy, DB executions, LLM calls, token estimates, latency, and memory statistics.
- Baselines and ablations implemented enough to run on fixtures even before the full datasets are downloaded.
- Clear TODOs only where external data, API credentials, or private benchmark access is required.
