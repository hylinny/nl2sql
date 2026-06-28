# Implementation Blueprint

## Target repository layout

```text
.
├── AGENTS.md
├── README.md
├── pyproject.toml
├── configs/
│   ├── default.yaml
│   ├── evo_sem_sql.yaml
│   ├── baselines.yaml
│   └── datasets.example.yaml
├── evo_sem_sql/
│   ├── __init__.py
│   ├── cli.py
│   ├── schema/
│   │   ├── introspect.py
│   │   ├── profile.py
│   │   └── graph_builder.py
│   ├── memory/
│   │   ├── records.py
│   │   ├── store.py
│   │   ├── graph.py
│   │   ├── retrieval.py
│   │   └── consolidation.py
│   ├── nl2sql/
│   │   ├── router.py
│   │   ├── prompt_builder.py
│   │   ├── generator.py
│   │   ├── templates.py
│   │   ├── guardrails.py
│   │   └── repair.py
│   ├── llm/
│   │   ├── base.py
│   │   ├── fake.py
│   │   └── openai_provider.py
│   ├── datasets/
│   │   ├── base.py
│   │   ├── spider.py
│   │   ├── spider2.py
│   │   ├── bird.py
│   │   └── bird_interact.py
│   ├── evaluation/
│   │   ├── metrics.py
│   │   ├── runner.py
│   │   ├── official.py
│   │   └── report.py
│   └── utils/
│       ├── logging.py
│       ├── token_estimator.py
│       └── paths.py
├── scripts/
│   ├── extract_odyssey.py
│   ├── build_lit_matrix.py
│   ├── prepare_datasets.py
│   ├── run_smoke.py
│   └── make_report.py
├── tests/
│   ├── fixtures/
│   │   └── toy_business.db
│   ├── test_schema_profile.py
│   ├── test_memory_store.py
│   ├── test_router.py
│   ├── test_guardrails.py
│   ├── test_templates.py
│   └── test_smoke_benchmark.py
├── research/
│   ├── raw/
│   │   └── odyssey.zip
│   ├── notion_exports/
│   └── extracted/
├── docs/
│   ├── literature_matrix.md
│   ├── progress_log.md
│   ├── methodology.md
│   ├── benchmark_results.md
│   └── paper_skeleton.md
└── artifacts/
    ├── memory/
    ├── runs/
    └── reports/
```

## Phase 0: repository bootstrap

Create:

- `pyproject.toml` with package metadata and dependencies.
- `README.md` with setup commands.
- `configs/*.yaml`.
- `docs/progress_log.md`.
- Basic package directories.

Suggested dependencies:

```toml
pydantic = "^2"
networkx = "^3"
pandas = "^2"
numpy = "^1"
sqlglot = "^25"
pyyaml = "^6"
typer = "^0.12"
rich = "^13"
rapidfuzz = "^3"
scikit-learn = "^1"
pytest = "^8"
pytest-cov = "^5"
```

Optional dependencies behind extras:

- `pymupdf` for PDF extraction.
- `openai` for real LLM provider.
- `sentence-transformers` or `faiss-cpu` for vector retrieval.

Unit tests must run with no API key.

## Phase 1: literature ingestion

Implement `scripts/extract_odyssey.py`:

Inputs:

```bash
python scripts/extract_odyssey.py --zip research/raw/odyssey.zip --out research/extracted
```

Outputs:

- `research/extracted/papers.jsonl`
- `research/extracted/text/<paper_stem>.txt`
- `research/extracted/manifest.json`
- `docs/literature_matrix.md`

Each paper record should include:

- filename,
- title guess,
- first-page text,
- abstract guess,
- detected keywords,
- likely category,
- notes/TODO summary.

Implement robust fallbacks:

- Skip `__MACOSX` files.
- Skip `.DS_Store`.
- Extract PDFs with PyMuPDF if installed; otherwise provide clear error.
- Continue on individual PDF failures and log them.

Initial categories:

- NL2SQL benchmark/dataset.
- NL2SQL prompting/reasoning.
- NL2SQL agent/workflow.
- Schema linking/semantic layer.
- Memory architecture.
- Agent self-improvement.
- General LLM/agent course material.

Implement `scripts/build_lit_matrix.py` to create a Markdown table with columns:

| Paper | Category | Core idea | Method components to reuse | Gap/limitation | Relevance to EvoSemSQL |

Codex should make best-effort summaries from extracted text. If no LLM key is configured, generate keyword-based preliminary summaries and mark them as preliminary.

## Phase 2: schema exploration

Implement these modules:

### `schema/introspect.py`

Functions:

- `introspect_sqlite(path: str) -> SchemaInfo`
- future extension point for DuckDB/Postgres.

`SchemaInfo` should include:

- tables,
- columns,
- types,
- primary keys,
- foreign keys,
- indexes where available.

### `schema/profile.py`

Functions:

- `profile_sqlite(path: str, sample_rows: int, max_distinct: int) -> DatabaseProfile`

Compute:

- row count,
- null fraction,
- approximate distinct count,
- min/max for numeric/date-like columns,
- bounded examples for text/categorical columns.

### `schema/graph_builder.py`

Build initial memory graph from schema and profiles.

Relationship inference:

- declared FK edges,
- name similarity between `*_id` columns,
- type-compatible columns,
- optional sample overlap under budget.

Every inferred edge must record confidence and evidence.

## Phase 3: memory graph and store

### `memory/records.py`

Define typed records:

- `MemoryNode`
- `MemoryEdge`
- `DatabaseNodePayload`
- `TableNodePayload`
- `ColumnNodePayload`
- `RelationNodePayload`
- `BusinessConceptPayload`
- `QueryIntentPayload`
- `SQLTemplatePayload`
- `AnswerFactPayload`
- `SkillPayload`
- `FailurePayload`
- `InteractionTrace`
- `RouteDecision`

### `memory/store.py`

Start with SQLite-backed persistence:

- `nodes(id, type, summary, payload_json, confidence, created_at, updated_at, last_used_at)`
- `edges(src, dst, type, payload_json, confidence, created_at)`
- `traces(id, query, route, sql, result_preview_json, metrics_json, created_at)`

APIs:

- `upsert_node(node)`
- `upsert_edge(edge)`
- `get_node(id)`
- `neighbors(id, edge_types=None, max_depth=1)`
- `search_nodes(query, types=None, top_k=20)`
- `save_trace(trace)`

Use SQLite FTS5 or a simple lexical index for the first version.

### `memory/retrieval.py`

Implement hybrid retrieval:

- lexical match over summaries, names, aliases;
- fuzzy match for table/column names;
- optional vector scoring if provider exists;
- graph expansion with type and budget constraints.

Output a compact `RetrievedSubgraph` containing only relevant nodes and edges.

### `memory/consolidation.py`

Implement:

- intent canonicalization,
- paraphrase linking,
- SQL template induction from successful SQL,
- answer fact insertion with TTL policy,
- failure node insertion on SQL errors,
- duplicate concept merge heuristic.

## Phase 4: NL2SQL runtime

### `nl2sql/router.py`

Routes:

- `DIRECT_MEMORY`
- `TEMPLATE_EXECUTION`
- `COMPACT_GENERATION`
- `FULL_SCHEMA_FALLBACK`
- `ASK_CLARIFICATION` for interactive use, but benchmark mode must avoid asking and instead mark ambiguous.

Decision features:

- best memory score,
- answer fact freshness,
- template slot coverage,
- subgraph coverage of query entities,
- linked failure nodes,
- ambiguity score.

### `nl2sql/templates.py`

Represent SQL templates as `sqlglot` AST or string with named slots.

Example:

```sql
SELECT {dimension}, SUM({metric}) AS value
FROM {table}
WHERE {date_col} BETWEEN :start_date AND :end_date
GROUP BY {dimension}
ORDER BY value DESC
LIMIT :k
```

Slot-filling must validate that selected columns exist and types are compatible.

### `nl2sql/prompt_builder.py`

Build two prompt types:

1. `compact_generation_prompt(query, subgraph, dialect)`
2. `full_schema_prompt(query, schema, dialect)`

Prompt must include:

- schema fragment,
- business concepts,
- known joins,
- relevant failures to avoid,
- output requirement: SQL only unless configured otherwise.

### `nl2sql/generator.py`

Provider abstraction:

- `FakeLLMProvider`: deterministic rule-based outputs for tests.
- `OpenAIProvider`: optional, requires API key.
- `ManualProvider`: reads SQL from fixture for debugging.

### `nl2sql/guardrails.py`

Use `sqlglot` to enforce:

- single statement,
- read-only SELECT/WITH only,
- no INSERT/UPDATE/DELETE/CREATE/DROP/ALTER/ATTACH/PRAGMA mutations,
- timeout/row limit at execution layer.

### `nl2sql/repair.py`

Bounded repair:

- max repair attempts from config,
- use error message and compact subgraph,
- log every attempt,
- never loop indefinitely.

## Phase 5: baselines

Implement a common interface:

```python
class NL2SQLMethod:
    def prepare_database(self, db_path: str) -> None: ...
    def answer(self, question: str, db_path: str, context: dict) -> MethodResult: ...
```

Baselines:

1. `FullSchemaBaseline`: prompt with entire schema each time.
2. `StaticSchemaRetrievalBaseline`: retrieve schema fragment but no evolving memory.
3. `ExactMemoBaseline`: exact normalized question cache only.
4. `FAQOnlyBaseline`: semantic answer cache without SQL templates or graph expansion.
5. `EvoSemSQL`: full method.

Ablations:

- `EvoSemSQLNoGraphExpansion`
- `EvoSemSQLNoTemplates`
- `EvoSemSQLNoAnswerCache`
- `EvoSemSQLNoFailureMemory`
- `EvoSemSQLNoFreshnessGate`
- `EvoSemSQLStaticMemory`

## Phase 6: evaluation runner

### CLI examples

```bash
python -m evo_sem_sql.cli index-db --db tests/fixtures/toy_business.db --memory artifacts/memory/toy.sqlite
python -m evo_sem_sql.cli ask --db tests/fixtures/toy_business.db --memory artifacts/memory/toy.sqlite --question "What were sales by region last month?"
python scripts/run_smoke.py
pytest -q
```

Benchmark examples:

```bash
python -m evo_sem_sql.cli benchmark \
  --dataset spider2 \
  --dataset-root "$SPIDER2_ROOT" \
  --method evo_sem_sql \
  --config configs/evo_sem_sql.yaml \
  --out artifacts/runs/spider2_evo.jsonl

python -m evo_sem_sql.cli compare \
  --runs artifacts/runs/*.jsonl \
  --out docs/benchmark_results.md
```

### Run log schema

Every evaluated question should produce one JSONL record:

```json
{
  "run_id": "...",
  "dataset": "spider2",
  "db_id": "...",
  "question_id": "...",
  "turn_id": 0,
  "question": "...",
  "method": "evo_sem_sql",
  "route": "COMPACT_GENERATION",
  "predicted_sql": "...",
  "execution_result_preview": [],
  "gold_sql_available": false,
  "execution_correct": null,
  "official_score": null,
  "llm_calls": 1,
  "prompt_tokens_est": 1300,
  "completion_tokens_est": 120,
  "schema_tokens_est": 500,
  "db_exec_count": 1,
  "latency_ms": 1450,
  "memory_nodes_before": 104,
  "memory_nodes_after": 108,
  "memory_hit": false,
  "stale_memory_error": false,
  "errors": []
}
```

## Phase 7: documentation and paper artifacts

Generate:

- `docs/literature_matrix.md`
- `docs/methodology.md`
- `docs/benchmark_results.md`
- `docs/ablation_plan.md`
- `docs/progress_log.md`
- `docs/paper_skeleton.md`

The paper skeleton should include:

1. Abstract.
2. Introduction.
3. Related Work.
4. Method.
5. Streaming NL2SQL Evaluation Protocol.
6. Experiments.
7. Ablations.
8. Analysis.
9. Limitations.
10. Conclusion.

## Minimum smoke test

Create a toy SQLite DB with:

- `customers(customer_id, region, signup_date, status)`
- `orders(order_id, customer_id, order_date, amount, status)`
- FK `orders.customer_id -> customers.customer_id`

Questions:

1. “What is total revenue by region?”
2. “Show sales by region.”
3. “Which region has the highest revenue?”
4. “How many active customers are there?”
5. “Count active customers.”

Expected behavior:

- Q1 likely uses compact generation.
- Q2 should hit intent/template memory from Q1.
- Q3 should reuse the same concept/template with top-1 modification.
- Q4 creates or uses business concept `active customer`.
- Q5 should direct-memory or template-hit depending on TTL policy.

## Implementation order for Codex

1. Create skeleton and tests.
2. Build toy DB fixture.
3. Implement schema introspection/profile.
4. Implement memory store and graph.
5. Implement retrieval and graph expansion.
6. Implement guardrails.
7. Implement fake LLM and prompt builders.
8. Implement router and templates.
9. Implement consolidation.
10. Implement smoke benchmark.
11. Implement Odyssey extraction.
12. Implement dataset adapters.
13. Implement benchmark report generation.
14. Run tests and update docs.
