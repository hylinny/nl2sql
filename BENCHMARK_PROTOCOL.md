# Benchmark Protocol

## Goal

Evaluate EvoSemSQL as an interactive, memory-evolving NL2SQL system. Accuracy alone is insufficient. The core evaluation must jointly measure correctness, LLM cost, schema-context size, database executions, latency, and memory safety.

## Datasets

Use these tiers:

### Tier 0: toy fixture

Always runnable. Used for unit tests, CI, and debugging.

### Tier 1: Spider 1.0 or small public NL2SQL sanity set

Used only as a smoke benchmark because it is less realistic for modern enterprise/database-agent workloads.

### Tier 2: Spider 2.0

Main benchmark for complex, realistic database tasks. Implement adapter and official-evaluator integration when dataset files/scripts are available.

### Tier 3: BIRD / BIRD-Interact

Main benchmark for business-like database QA and interactive behavior. Implement both single-turn and interactive adapters if files are present.

### Tier 4: user-defined business DBs

Optional private evaluation where EvoSemSQL is expected to shine because repeated analytical intents naturally occur.

## Evaluation modes

### Cold one-shot mode

Memory is bootstrapped from schema/profile only. It is reset for each question or each database depending on config.

Purpose: compare against standard one-shot NL2SQL.

### Online streaming mode

Memory is bootstrapped once per database and updated after each interaction. Questions are grouped by database and processed in a fixed order.

Purpose: measure whether memory improves over time.

### Strict online mode

The system may update memory only from its own generated SQL, execution results, errors, and validation signals. It may not insert gold SQL or gold answers into memory.

This is the main scientific setting.

### Oracle warm-start mode

Gold SQL or gold schema links may be used to seed memory before evaluation. This is an upper bound only and must be labeled clearly.

## Streaming workload construction

For datasets with natural interactions, preserve the provided interaction/session order.

For single-turn datasets:

1. Group questions by `db_id`.
2. Sort by original dataset order unless an explicit timestamp/session id exists.
3. Process each group as a stream.
4. Report performance by turn index bucket: early, middle, late.
5. Optionally create paraphrase/repetition stress tests, but keep them separate from the main benchmark.

### Repetition/paraphrase stress test

Because many public NL2SQL datasets underrepresent repeated business questions, create a transparent auxiliary benchmark:

- Start from gold or validated predicted SQL templates.
- Generate or manually define paraphrases for common intent families.
- Vary slots such as date ranges, top-k, dimensions, and metrics.
- Evaluate whether the system reuses memory safely.

This test should never be presented as the official dataset score.

## Metrics

### Accuracy metrics

- Execution accuracy where gold result or official evaluator is available.
- Test-suite accuracy if official test-suite evaluator exists.
- Exact SQL match only as a secondary diagnostic.
- Valid SQL rate.
- Result-shape correctness when full gold result is unavailable.

### Efficiency metrics

- LLM calls per question.
- Prompt tokens per question.
- Completion tokens per question.
- Schema tokens per question.
- Estimated cost per question.
- Database executions per question.
- Latency per question.
- Repair attempts per question.

### Memory metrics

- Memory hit rate.
- Direct-answer hit rate.
- Template hit rate.
- Compact-generation route rate.
- Full-schema fallback route rate.
- Memory hit precision.
- Stale-memory error rate.
- Number of nodes/edges over time.
- Memory growth per 100 interactions.
- Average retrieved subgraph size.

### Robustness metrics

- SQL error rate.
- Empty-result suspiciousness rate.
- Ambiguous-query failure rate.
- Wrong-join failure rate.
- Hallucinated-column rate.

## Baselines

1. **FullSchemaBaseline**: Every query gets the full schema.
2. **StaticSchemaRetrievalBaseline**: Retrieves relevant schema snippets, but memory does not evolve.
3. **ExactMemoBaseline**: Caches normalized exact user questions and answers.
4. **FAQOnlyBaseline**: Uses semantic question-answer retrieval but no SQL templates, graph, or freshness-aware routing.
5. **TemplateOnlyBaseline**: Reuses SQL templates but has no semantic memory graph.
6. **EvoSemSQL**: Full proposed method.

## Ablations

- No graph expansion.
- No answer cache.
- No SQL template memory.
- No failure memory.
- No freshness/staleness gate.
- No business concept nodes.
- Static memory only: bootstrap memory but no online updates.
- Unlimited full-schema fallback disabled/enabled.

## Main result tables

### Table 1: Main benchmark accuracy and efficiency

| Dataset | Method | Exec Acc | Valid SQL | LLM Calls/Q | Prompt Tok/Q | Schema Tok/Q | DB Exec/Q | Latency/Q | Cost/Q |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|

### Table 2: Online learning curve

| Dataset | Method | Turns 1-5 Acc | Turns 6-20 Acc | Turns 21+ Acc | Memory Hit Rate | Full Fallback Rate |
|---|---:|---:|---:|---:|---:|---:|

### Table 3: Ablation

| Dataset | Variant | Exec Acc | Prompt Tok/Q | DB Exec/Q | Stale Error | Notes |
|---|---:|---:|---:|---:|---:|---|

### Table 4: Memory safety

| Dataset | Direct Hits | Correct Direct Hits | Hit Precision | Stale Errors | Unsafe Cache Prevented |
|---|---:|---:|---:|---:|---:|

## Required commands

### Unit tests

```bash
pytest -q
```

### Smoke benchmark

```bash
python scripts/run_smoke.py --out artifacts/runs/smoke.jsonl
python -m evo_sem_sql.cli compare --runs artifacts/runs/smoke.jsonl --out docs/benchmark_results.md
```

### Spider 2.0

```bash
python -m evo_sem_sql.cli benchmark \
  --dataset spider2 \
  --dataset-root "$SPIDER2_ROOT" \
  --method evo_sem_sql \
  --mode online_strict \
  --config configs/evo_sem_sql.yaml \
  --out artifacts/runs/spider2_evo_online.jsonl
```

### BIRD

```bash
python -m evo_sem_sql.cli benchmark \
  --dataset bird \
  --dataset-root "$BIRD_ROOT" \
  --method evo_sem_sql \
  --mode online_strict \
  --config configs/evo_sem_sql.yaml \
  --out artifacts/runs/bird_evo_online.jsonl
```

### BIRD-Interact

```bash
python -m evo_sem_sql.cli benchmark \
  --dataset bird_interact \
  --dataset-root "$BIRD_INTERACT_ROOT" \
  --method evo_sem_sql \
  --mode online_strict \
  --config configs/evo_sem_sql.yaml \
  --out artifacts/runs/bird_interact_evo_online.jsonl
```

## Fairness rules

- Use the same LLM model and temperature for all LLM-based methods unless testing model sensitivity.
- Count all repair calls as LLM calls.
- Count SQL executions used for exploration, verification, and repair.
- Keep schema sampling budgets identical across methods unless the method explicitly does not use exploration.
- Do not use gold SQL during strict online evaluation.
- Separate official benchmark results from synthetic/paraphrase stress-test results.

## Reporting rules

For every table, include:

- dataset version,
- split,
- number of examples,
- model/provider,
- config path,
- command run,
- date/time,
- git commit hash if available,
- number of failed/skipped examples and why.

## Expected first milestone

The first implementation milestone does not need full Spider 2.0 scores. It must show:

- all unit tests passing,
- smoke benchmark table generated,
- missing-dataset checks for Spider 2.0/BIRD/BIRD-Interact,
- documented commands to run once datasets are available,
- preliminary literature matrix from Odyssey papers.
