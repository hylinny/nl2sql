# AGENTS.md

## Role

You are implementing a research-grade NL2SQL system for a top-conference submission. Treat this repository as a scientific artifact: every claim must be backed by code, logs, experiments, or clearly marked TODOs.

## Project name

Use the working name **EvoSemSQL** unless the user later renames it.

## Main objective

Build and benchmark a self-evolving semantic memory layer for interactive NL2SQL. The system should reduce repeated schema-prompting, unnecessary SQL executions, LLM calls, and cost while preserving or improving execution accuracy.

## Non-negotiable requirements

- Do not only write a plan. Implement runnable code, tests, commands, and generated documentation.
- Do not assume benchmark datasets are present. Implement adapters that detect missing datasets and fail with helpful setup instructions.
- Do not require paid API calls for unit tests. Provide a deterministic fake LLM provider and optional real providers.
- Do not hard-code absolute paths or secrets.
- Never execute destructive SQL. Only allow read-only SQL in evaluation and demos.
- Log every LLM call, prompt token estimate, SQL execution, memory hit, routing decision, and failure.
- Preserve experiment reproducibility with config files and JSONL run logs.
- For any result table, include the exact command and config that produced it.

## Preferred implementation stack

- Python 3.10+
- `pytest` for tests
- `pydantic` or dataclasses for typed records
- `sqlglot` for SQL parsing and read-only validation
- SQLite for the first memory store
- `networkx` for the semantic memory graph prototype
- Local lexical retrieval by default; optional embedding provider behind an interface
- `typer` or `argparse` for CLIs
- JSONL for traces
- Markdown for generated research notes

## Repository workflow

1. Read `RESEARCH_METHODOLOGY.md`, `IMPLEMENTATION_BLUEPRINT.md`, and `BENCHMARK_PROTOCOL.md` first.
2. Create the repo skeleton.
3. Implement the smallest runnable system before adding sophistication.
4. Add tests for each module as it is built.
5. Run tests and smoke benchmarks before reporting completion.
6. Keep a running `docs/progress_log.md` containing completed tasks, commands run, and known limitations.

## Coding expectations

- Make functions small and typed.
- Prefer explicit records over loose dictionaries at boundaries.
- Write docstrings for public modules.
- Add defensive checks for missing files, missing API keys, invalid SQL, and unsupported datasets.
- Use deterministic seeds for experiments.
- Produce human-readable errors.

## Research expectations

The system must support these comparisons:

- Full-schema NL2SQL baseline.
- Static schema retrieval baseline.
- Exact query memoization baseline.
- FAQ-only RAG baseline.
- EvoSemSQL full method.
- Ablations removing graph expansion, memory consolidation, staleness gating, template reuse, and failure memory.

The benchmark must track:

- Execution accuracy or dataset-specific official score.
- Valid SQL rate.
- LLM calls per question.
- Prompt and completion token estimates.
- Schema tokens included per query.
- Number of database executions per question.
- End-to-end latency.
- Memory hit rate and memory hit precision.
- Stale-memory error rate.
- Memory size growth.

## Completion response format

When you finish a task, report:

- Files changed.
- Commands run.
- Tests passed or failed.
- Benchmark smoke results.
- Limitations and next tasks.

Do not claim full Spider 2.0/BIRD/BIRD-Interact performance unless the datasets were actually present and the benchmark commands were actually run.
