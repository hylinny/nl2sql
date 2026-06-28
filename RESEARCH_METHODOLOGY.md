# Research Methodology: EvoSemSQL

## Working title

**EvoSemSQL: Self-Evolving Semantic Memory for Cost-Aware Interactive NL2SQL**

## Problem

State-of-the-art NL2SQL systems often solve each query as if it were isolated. They repeatedly pass large schemas, re-discover column semantics, regenerate similar SQL, and execute exploratory queries even when a database has already been used many times. This is especially wasteful in business analytics settings where users repeatedly ask variants of the same questions over stable schemas.

The research problem is to design an NL2SQL agent that improves over the lifetime of a database and workload without leaking gold labels or blindly trusting stale cached answers.

## Central hypothesis

A typed, self-evolving semantic memory graph can reduce LLM context size, LLM calls, and database executions while preserving or improving execution accuracy, because many future queries can be resolved through verified business concepts, SQL templates, schema-linking traces, and prior validated query plans.

## Core contribution

EvoSemSQL is a cost-aware interactive NL2SQL architecture with four new components:

1. **Typed Semantic Memory Graph**: a graph of databases, tables, columns, relationships, business concepts, query intents, SQL templates, answer facts, skills, and failure cases.
2. **Answerability and Freshness Gate**: a routing gate that decides whether a query can be answered from memory, needs a template execution, needs compact-context SQL generation, or must fall back to full-schema generation.
3. **Self-Evolving Consolidation Loop**: successful and failed interactions update memory through deduplication, template induction, concept strengthening, stale fact invalidation, and failure-case storage.
4. **Efficiency-Centered NL2SQL Benchmarking**: evaluation on streaming workloads where accuracy is measured jointly with LLM calls, schema tokens, database executions, latency, and memory errors.

## System overview

```text
                 ┌────────────────────────┐
                 │ Offline DB Exploration │
                 └───────────┬────────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │ Semantic Memory     │
                  │ Graph + Stores      │
                  └──────────┬──────────┘
                             │
User query ──► Query Understanding ──► Retrieval + Graph Expansion
                             │
                             ▼
                 Answerability/Freshness Router
              ┌──────────────┼────────────────┐
              ▼              ▼                ▼
       Direct memory    SQL template      Compact-context
       answer           instantiation     SQL generation
              │              │                │
              └──────────────┴──────┬─────────┘
                                     ▼
                          Guarded SQL execution
                                     │
                                     ▼
                           Validation + Repair
                                     │
                                     ▼
                         Memory update/consolidation
```

## Memory graph schema

### Node types

- `DatabaseNode`: database identifier, dialect, domain summary, version/snapshot metadata.
- `TableNode`: table name, natural-language description, row count, primary keys, volatility estimate.
- `ColumnNode`: column name, type, null rate, distinct count, examples, semantic aliases, unit hints.
- `RelationNode`: foreign key, inferred join, join confidence, evidence.
- `BusinessConceptNode`: domain concept such as “active customer,” “gross merchandise value,” “churned user,” or “net revenue.”
- `QueryIntentNode`: canonicalized natural-language intent, paraphrases, required slots, expected answer type.
- `SQLTemplateNode`: parameterized SQL AST/template, slots, dialect, required tables/columns, constraints.
- `AnswerFactNode`: cached result for a stable or TTL-bounded query, provenance, source SQL, snapshot id.
- `SkillNode`: reusable procedure such as “date range normalization,” “top-k by metric,” “cohort retention,” or “ambiguous ID resolution.”
- `FailureNode`: failed SQL, error message, bad schema link, or wrong assumption to avoid.

### Edge types

- `has_table`, `has_column`, `foreign_key`, `inferred_join`
- `describes`, `alias_of`, `unit_of`, `derived_from`
- `uses_table`, `uses_column`, `requires_join`
- `answers`, `instantiates`, `paraphrase_of`
- `supersedes`, `invalidates`, `contradicts`
- `failed_because`, `avoid_when`

### Memory item requirements

Every memory item must have:

- stable id,
- type,
- natural-language summary,
- structured payload,
- provenance,
- confidence score,
- creation time,
- last-used time,
- optional TTL or snapshot dependency,
- source trace ids.

## Offline bootstrap

The offline bootstrap stage creates initial memory without seeing benchmark gold labels.

1. Introspect schema: tables, columns, types, keys, indexes, views.
2. Profile bounded samples: row counts, null rates, distinct counts, min/max for numeric/date columns, representative categorical values.
3. Infer relationships: declared FKs, name-based joins, value-overlap joins under sampling limits.
4. Summarize columns: generate low-cost semantic summaries from names, types, neighboring columns, and safe examples.
5. Build initial graph: database/table/column/relation nodes.
6. Seed concept candidates: detect common measures, dimensions, temporal fields, IDs, status flags, monetary columns, geographic columns.
7. Store provenance: every inferred relationship or concept must record evidence and confidence.

### Bootstrap pseudocode

```python
def bootstrap_memory(db):
    schema = introspect_schema(db)
    profiles = profile_database(db, sample_budget=CONFIG.sample_budget)
    fk_edges = extract_declared_foreign_keys(schema)
    inferred_edges = infer_join_edges(schema, profiles)
    concept_nodes = propose_business_concepts(schema, profiles, fk_edges + inferred_edges)
    graph = build_semantic_graph(schema, profiles, fk_edges, inferred_edges, concept_nodes)
    persist(graph)
    return graph
```

## Query-time routing

For each user query `q`, EvoSemSQL performs:

1. **Query normalization**: detect time ranges, metrics, entities, aggregation type, output form, and ambiguity.
2. **Hybrid retrieval**: retrieve candidate memory nodes using lexical similarity, optional embedding similarity, and schema-name matching.
3. **Graph expansion**: expand from retrieved nodes to adjacent tables, columns, joins, SQL templates, and business concepts.
4. **Answerability gate**:
   - route `DIRECT_MEMORY` if a cached answer fact is semantically equivalent, fresh, and supported by provenance;
   - route `TEMPLATE_EXECUTION` if a validated SQL template can be instantiated with the query slots;
   - route `COMPACT_GENERATION` if a compact subgraph contains enough schema and business semantics;
   - route `FULL_SCHEMA_FALLBACK` if retrieval confidence is low or ambiguity is high.
5. **Guarded execution**: validate read-only SQL, execute with row/time limits, capture errors.
6. **Validation/repair**: compare result type to expectation, catch empty-result pitfalls, perform bounded repair.
7. **Memory update**: write successful traces, templates, paraphrases, new concepts, and failure cases.

### Routing pseudocode

```python
def answer_query(query, db, memory):
    norm = normalize_query(query)
    seeds = retrieve_memory(norm, memory)
    subgraph = expand_graph(seeds, memory, budget=CONFIG.context_budget)
    decision = answerability_gate(norm, subgraph, db.snapshot_id)

    if decision.route == "DIRECT_MEMORY":
        result = load_cached_answer(decision.answer_fact_id)
        trace = make_trace(query, decision, sql=None, result=result)
        return result, update_usage(memory, trace)

    if decision.route == "TEMPLATE_EXECUTION":
        sql = instantiate_template(decision.template_id, norm.slots)
    elif decision.route == "COMPACT_GENERATION":
        prompt = build_compact_prompt(query, subgraph)
        sql = generate_sql(prompt)
    else:
        prompt = build_full_schema_prompt(query, db.schema)
        sql = generate_sql(prompt)

    sql = enforce_read_only(sql)
    result = execute_with_limits(db, sql)
    verdict = validate_result(query, sql, result, subgraph)

    if verdict.needs_repair:
        sql, result, verdict = bounded_repair(query, sql, result, verdict, subgraph)

    trace = make_trace(query, decision, sql=sql, result=result, verdict=verdict)
    consolidate_memory(memory, trace)
    return result, trace
```

## Answerability and freshness gate

The gate must be conservative. Returning an old answer is only allowed when all of these conditions hold:

- semantic equivalence between current query and cached query is above threshold;
- cached answer has provenance and source SQL;
- source tables are classified as stable or the item TTL has not expired;
- database snapshot id matches, or the changed tables do not affect the source SQL;
- expected answer type matches;
- no contradiction/failure node is linked to the candidate.

Otherwise, the system should reuse the memory item as context or template but re-execute SQL.

## Memory consolidation

After each successful query:

1. Canonicalize intent.
2. Link query to used tables, columns, joins, concepts, and SQL AST.
3. If the query resembles an existing intent, add it as a paraphrase and update frequency/confidence.
4. If the SQL can be parameterized, induce or update a SQL template.
5. If the result is safe to cache, create or refresh an answer fact with TTL/snapshot metadata.
6. If generation failed, create a failure node so retrieval can avoid the same mistake later.
7. Periodically merge duplicate concepts and retire stale low-confidence memory.

## Why this is different from simple caching

Exact cache:

- only catches identical strings,
- cannot explain schema semantics,
- cannot generalize to paraphrases,
- can return stale facts.

EvoSemSQL:

- stores typed provenance-rich semantic memory,
- generalizes via intents and templates,
- routes through conservative answerability gates,
- still executes SQL when values may be stale,
- learns schema links and business concepts over time.

## Why this is different from static schema RAG

Static schema RAG retrieves table and column descriptions but usually does not update from solved interactions. EvoSemSQL adds:

- SQL trace memory,
- template induction,
- failure memory,
- FAQ/answer memory with freshness checks,
- graph-level concept consolidation,
- efficiency-centered evaluation.

## Primary experimental questions

1. Does evolving memory improve execution accuracy over a full-schema baseline under context constraints?
2. Does it reduce LLM prompt tokens and number of LLM calls per query?
3. Does it reduce database executions through direct memory answers or template reuse?
4. How often does memory cause stale or wrong answers?
5. Which memory components matter most?
6. How quickly does performance improve as interactions accumulate?

## Expected claims if experiments succeed

- EvoSemSQL maintains competitive execution accuracy while substantially reducing average schema tokens and LLM calls.
- Template and intent memory help most on repeated business-query families.
- Graph expansion improves schema-linking precision over flat retrieval.
- Conservative freshness gating prevents most harmful stale-cache failures.
- Failure memory reduces repeated invalid SQL generation on difficult schemas.

## High-risk points

- Public benchmarks may not contain enough natural repetition to show memory benefits. Mitigation: evaluate both original interactive datasets and a transparent streaming/paraphrase protocol.
- Caching answers can be unsafe for volatile data. Mitigation: cache only snapshot-stable or TTL-bounded facts and report stale-error metric.
- LLM costs may be hard to compare across models. Mitigation: log provider-independent token estimates plus provider-specific cost when configured.
- Full Spider 2.0 evaluation may require external services or official scripts. Mitigation: implement adapters and clear setup checks, with toy smoke tests always runnable.

## Minimum viable paper narrative

The paper should not claim “we beat all SOTA NL2SQL.” A stronger and more plausible AAAI-style narrative is:

> Existing NL2SQL benchmarks emphasize one-shot accuracy, but deployed analytical agents operate over repeated database-specific workloads. We introduce a memory-evolving NL2SQL agent and an evaluation protocol that measures accuracy, context efficiency, query efficiency, and memory safety over streaming interactions.
