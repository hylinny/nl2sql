# Paper Skeleton: EvoSemSQL

## Title

EvoSemSQL: Self-Evolving Semantic Memory for Cost-Aware Interactive NL2SQL

## Abstract draft

Large-language-model NL2SQL systems often process each question independently, repeatedly prompting over large schemas and regenerating similar SQL for recurring analytical intents. This one-shot framing mismatches deployed business intelligence settings, where users ask related questions over stable databases and where cost, latency, and database load matter alongside accuracy. We propose EvoSemSQL, a self-evolving semantic memory layer for interactive NL2SQL. EvoSemSQL builds a typed graph of database semantics, business concepts, SQL templates, answer facts, and failure traces; routes queries through conservative answerability and freshness gates; and updates memory from validated interactions. We introduce an evaluation protocol that measures execution accuracy together with LLM calls, schema-context size, database executions, latency, and stale-memory errors over streaming workloads. Experiments on toy, Spider 2.0, BIRD, and BIRD-Interact settings should test whether evolving memory preserves accuracy while reducing repeated computation.

## 1. Introduction

Key points:

- NL2SQL has progressed rapidly, but most evaluation is one-shot.
- Real business workloads are repeated, evolving, and cost-sensitive.
- Full-schema prompting is expensive and often unnecessary.
- Static schema retrieval does not learn from solved interactions.
- Naive caching is unsafe because data can become stale and paraphrases differ.
- EvoSemSQL proposes typed, provenance-rich memory with safe routing.

Claim framing:

> We do not argue that memory replaces SQL generation. We argue that an NL2SQL agent should learn a database-specific semantic layer over time and use it to decide when SQL generation is necessary.

## 2. Related Work

Subsections:

1. Text-to-SQL benchmarks and datasets.
2. LLM prompting and reasoning for NL2SQL.
3. Schema linking and semantic layers.
4. Agentic NL2SQL workflows and repair.
5. Long-term memory for LLM agents.
6. Self-evolving/self-improving agents.
7. Caching, materialized views, and query reuse in databases.

Use `docs/literature_matrix.md` to populate this section.

## 3. Problem Formulation

Define a stream of questions over database `D`:

```text
S_D = [(q_1, t_1), ..., (q_n, t_n)]
```

A method outputs either:

- direct answer `a_i`,
- SQL `s_i` and answer `a_i`,
- abstention/clarification in interactive mode.

The objective is multi-objective:

```text
maximize accuracy
minimize LLM_calls, prompt_tokens, schema_tokens, DB_execs, latency, stale_errors
```

Memory state evolves:

```text
M_i = update(M_{i-1}, q_i, route_i, s_i, result_i, validation_i)
```

Strict evaluation forbids updating `M` from gold SQL/labels.

## 4. Method

### 4.1 Offline database exploration

Describe schema introspection, profiling, relationship inference, initial concept seeding.

### 4.2 Typed semantic memory graph

Describe node/edge schema and provenance.

### 4.3 Query routing

Describe direct memory, template execution, compact generation, full fallback.

### 4.4 Answerability/freshness gate

Explain conservative cache use and snapshot/TTL checks.

### 4.5 Memory consolidation

Explain successful trace insertion, paraphrase merging, SQL template induction, failure memory, forgetting.

## 5. Streaming NL2SQL Evaluation Protocol

Motivate why one-shot metrics are insufficient.

Define:

- cold mode,
- online strict mode,
- oracle warm-start upper bound,
- paraphrase/repetition stress test.

Metrics:

- execution accuracy,
- valid SQL,
- LLM calls,
- prompt/schema tokens,
- DB executions,
- latency,
- memory hit precision,
- stale errors.

## 6. Experiments

Datasets:

- toy fixture,
- Spider 2.0,
- BIRD,
- BIRD-Interact,
- optional Spider 1.0 sanity.

Baselines:

- full schema,
- static schema retrieval,
- exact memo,
- FAQ-only,
- template-only,
- EvoSemSQL.

## 7. Results

Expected tables are defined in `BENCHMARK_PROTOCOL.md`.

Important plots:

- accuracy vs turn index,
- prompt tokens vs turn index,
- memory hit rate vs turn index,
- DB executions vs turn index,
- stale errors by method.

## 8. Ablation Study

Ablations:

- no graph expansion,
- no templates,
- no answer cache,
- no failure memory,
- no freshness gate,
- static memory only.

## 9. Analysis

Qualitative examples:

1. Memory correctly resolves an ambiguous business concept.
2. Template reuse avoids full SQL generation.
3. Freshness gate prevents a stale answer.
4. Failure memory prevents repeated hallucinated joins.
5. Retrieval failure causes fallback.

## 10. Limitations

- Public datasets may not fully represent business repetition.
- Memory introduces new failure modes.
- Schema changes and data drift require robust invalidation.
- Some official benchmarks may restrict execution or labels.
- Top raw one-shot accuracy may still require large external models.

## 11. Conclusion

Emphasize EvoSemSQL as a step toward database agents that learn a reliable semantic layer over repeated use, optimizing not only correctness but also cost, latency, and database load.
