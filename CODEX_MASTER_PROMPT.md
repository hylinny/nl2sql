You are working in the root of my EvoSemSQL repository. Read these files first and treat them as binding project context:

- `AGENTS.md`
- `RESEARCH_METHODOLOGY.md`
- `IMPLEMENTATION_BLUEPRINT.md`
- `BENCHMARK_PROTOCOL.md`
- `PAPER_SKELETON.md`

Project objective: implement a research-grade prototype for **EvoSemSQL: Self-Evolving Semantic Memory for Cost-Aware Interactive NL2SQL**. The goal is a top-conference-quality methodology and benchmark harness, not just a demo.

Important local materials:

- If present, extract and analyze `research/raw/odyssey.zip`. It contains papers relevant to NL2SQL, schema linking, memory agents, and self-evolving agents. Build `docs/literature_matrix.md` from it.
- If present, read `research/notion_exports/` for my notes and incorporate useful points into `docs/methodology.md` and `docs/progress_log.md`.
- If present, read `docs/deep_research_synthesis.md` and reconcile it with the methodology.

Execute the project in phases. Do not stop after writing a plan. Implement runnable code and tests.

Phase 0 — bootstrap repository:
1. Create `pyproject.toml`, package directories, configs, README, and docs folders according to `IMPLEMENTATION_BLUEPRINT.md`.
2. Add deterministic test fixtures and a toy SQLite business database.
3. Make sure `pytest -q` can run without API keys.

Phase 1 — Odyssey/literature ingestion:
1. Implement `scripts/extract_odyssey.py` to extract PDFs from `research/raw/odyssey.zip`, skipping `__MACOSX` and `.DS_Store`.
2. Produce `research/extracted/papers.jsonl`, extracted text files, and `research/extracted/manifest.json`.
3. Implement `scripts/build_lit_matrix.py` to generate `docs/literature_matrix.md` with categories, core ideas, reusable components, gaps, and relevance to EvoSemSQL.
4. LLM API key (OpenAI) provided in .env file. If no LLM key is available, use deterministic keyword/section-based summaries and mark them preliminary.

Phase 2 — core system:
1. Implement SQLite schema introspection and profiling.
2. Implement typed memory records, SQLite-backed memory store, and NetworkX semantic graph.
3. Implement lexical/fuzzy retrieval and graph expansion.
4. Implement SQL read-only guardrails with `sqlglot`.
5. Implement OpenAI provider (preferred, api key in .env) or fake LLM provider.
6. Implement query router with routes: `DIRECT_MEMORY`, `TEMPLATE_EXECUTION`, `COMPACT_GENERATION`, `FULL_SCHEMA_FALLBACK`.
7. Implement SQL template induction/reuse and memory consolidation.
8. Implement failure memory.

Phase 3 — baselines and ablations:
1. Implement `FullSchemaBaseline`, `StaticSchemaRetrievalBaseline`, `ExactMemoBaseline`, `FAQOnlyBaseline`, `TemplateOnlyBaseline`, and `EvoSemSQL` behind one common interface.
2. Implement ablations: no graph expansion, no templates, no answer cache, no failure memory, no freshness gate, static memory only.

Phase 4 — evaluation harness:
1. Implement dataset adapters for toy, Spider 1.0 if easy, Spider 2.0, BIRD, and BIRD-Interact.
2. If external datasets are missing, fail with clear setup instructions rather than crashing.
3. Implement benchmark modes: cold one-shot, online strict, oracle warm-start upper bound, paraphrase/repetition stress test.
4. Log JSONL traces with accuracy, valid SQL, LLM calls, prompt token estimates, schema token estimates, DB executions, latency, route, memory hits, stale-memory errors, and memory size.
5. Implement `compare` report generation to `docs/benchmark_results.md`.

Phase 5 — smoke experiments:
1. Run `pytest -q`.
2. Run the toy smoke benchmark.
3. Generate `docs/benchmark_results.md` and `docs/progress_log.md`.
4. Verify that the toy benchmark demonstrates compact generation first, then template or memory reuse on paraphrased repeated questions.

Phase 6 — research docs:
1. Generate or update `docs/methodology.md` with the final architecture and algorithms.
2. Generate or update `docs/paper_skeleton.md` based on `PAPER_SKELETON.md`.
3. Add `docs/ablation_plan.md` and a clear `docs/next_experiments.md` for Spider 2.0/BIRD/BIRD-Interact once datasets and API keys are configured.

Hard constraints:

- Never use gold SQL or gold answers to update memory in strict online mode.
- Never execute destructive SQL.
- Unit tests can require internet or paid API calls, use the provided OPENAI_API_KEY without holding back.
- Make all missing dataset/API situations explicit and recoverable.
- Record exact commands run in `docs/progress_log.md`.
- Do not claim benchmark performance unless the commands were actually run.

At the end, report:

1. Files created/changed.
2. Commands run and outputs.
3. Test results.
4. Smoke benchmark results.
5. Current limitations.
6. The exact next command I should run to benchmark Spider 2.0, BIRD, and BIRD-Interact after placing dataset roots in the environment.
