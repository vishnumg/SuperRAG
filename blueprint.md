# SuperRAG — Provider‑Agnostic, Class‑First RAG Library

**Tagline:** One Python library for everything RAG—from lean, LLM‑free retrieval to agentic, multimodal, cache‑first pipelines. No servers, no SaaS. **Works with any LLM, embedding model, or vector database via adapters.**

---

## 1) Principles

* **Class‑first & pluggable:** All capabilities are small, subclassable `Base*` components.
* **Provider‑agnostic:** No provider is hard‑coded; all external services connect through adapters.
* **LLM‑optional:** Retrieval stays IR‑pure; only components that need LLMs accept them.
* **Agentic by design:** Planning, multi‑hop retrieval, active lookups, routing, memory, termination.
* **Managed‑DB native:** Works with hosted and self‑hosted vector/sparse stores through drivers.
* **Production‑minded:** caching, grounding, guardrails, evaluation, tracing, governance.

---

## 2) Core Object Model (what you subclass)

**Load:** `BaseLoader` (FS, Git, Web, DB)

**Chunk:** `BaseChunker` (Semantic, Structured, Sliding, Adaptive, Code, Table, Transcript, Layout/PDF)

**Context:** `BaseContextualizer` (prefix summaries, breadcrumbs, entities)

**Embed:** `BaseEmbedder` (text/code/image) — any embedding model via adapters

**Indexing:** `BaseDenseIndex`, `BaseSparseIndex`, `BaseMultiVectorIndex`
• Local: in‑process ANN
• Managed/self‑hosted: driver adapters (see §12)

**Retrieval:** `BaseRetriever` (DocRetriever, ChunkRetriever, TwoStageRetriever, HybridRetriever, FusionRetriever)

**Rerank:** `BaseReranker` (CrossEncoder, LLM‑Reranker, Heuristic/MMR)

**Curate:** `BaseCurator` (MMR/diversity, de‑dup, per‑doc caps, budgeted context packing)

**Generate:** `BaseGenerator` (LLMGenerator, ExtractiveReader)

**Guard:** `BaseGuardrail` (Grounding, PII Redaction, Policy)

**Plan/Agent:** `BasePlanner`, `BaseAgentMemory`, `BaseTerminationPolicy`
(ReAct, IRCoT, Plan‑DAG; short/episodic memory; hop/cost/coverage termination)

**Cache:** `BaseCache` (In‑Memory, Redis‑style, SQL‑style)

**Eval:** `BaseEvaluator` (IR, RAG grounding, agentic metrics, latency/cost)

**Trace:** `BaseTracer` (Simple, OTEL)

**Route:** `BaseRouter` (MetadataRouter, CostAwareRouter, RetrievalGate)

> All bases expose small typed methods and lifecycle hooks (`configure`, `close`).

---

## 3) Provider‑Agnostic Chat Abstraction

**Class:** `llm.BaseChatModel` (RAG‑native)

* Unified interface used by Generator, Planner, LLM‑Reranker, Judge, Vision‑Generator.
* Supports: tool/function calling, streaming, structured/JSON outputs, multimodal parts (image/table), token usage + budgets, tracing.
* Capability probes: `TOOL_CALLS`, `VISION`, `JSON_MODE`, `STREAMING`, `LOGPROBS`, `BATCH`, `SYSTEM_PROMPT`.
* Adapters: generic `ChatModelAdapter` objects for any vendor/runtime.
* **LLM roles are explicit:** `generator`, `rewriter` (HyDE/Multi‑Query), `reranker_llm`, `judge`, `vision_llm`. Missing roles quietly disable dependent features (graceful) or raise (strict).

**Guarantees:** `search()` never invokes an LLM. Only LLM‑accepting components call models.

---

## 4) Pipelines (compose instances)

* **SearchPipeline (LLM‑free):** Retriever → Reranker → Curator → (RetrievalCache, Trace)
* **AskPipeline (LLM‑optional):** SearchPipeline → Generator → Guardrails → (SynthesisCache)
* **AgenticPipeline:** Planner ↔ SearchPipeline (multi‑hop loop) → Generator → Guardrails → TerminationPolicy → Memory

**Capability modes**
• *Graceful* (default): skip missing‑LLM features, log in trace.
• *Strict*: raise if an LLM‑dependent component lacks an LLM.

---

## 5) Chunking & Enrichment

* **Semantic (default):** topic‑aware boundaries; token budgets; overlap 10–20%.
* **Structured:** headings/tables/code preserved; breadcrumb metadata.
* **Adaptive:** domain‑aware sizes (legal clauses, functions/classes, dialogue turns).
* **Layout‑aware:** PDFs (sections, figures, captions), page/region metadata.
* **Transcript‑aware:** speaker turns, timecodes.
* **Contextualizers:** micro‑summaries, entities, titles/sections prepended for recall lift.

---

## 6) Indexing & Vector/Sparse Stores

* **Drivers:** generic adapters implement `BaseDenseIndex` / `BaseSparseIndex` / `BaseMultiVectorIndex`.
* **Hybrid retrieval:**

  * *Native* when store supports hybrid (vector + lexical in one query).
  * *Logical* fusion in SuperRAG (vector search + BM25 → z‑score/weighted rank fusion).
* **Two‑layer indexing:** doc‑level summaries for coarse select → chunk‑level for precision; restrict chunk search by doc IDs.
* **Multi‑vector (late‑interaction):** native when available; else emulated via sibling collections + client‑side MaxSim (bounded tokens).
* **Metadata filters:** normalized schema (tenant/project/type/date/tags/lang) mapped transparently to store DSLs.
* **Ops:** batch upsert with backoff; versioned records (`data_version`); dedupe; warmup queries; snapshots/exports; cross‑store migrator.

---

## 7) Retrieval Defaults (that win)

* **TwoStageRetriever:** doc→chunk.
* **HybridRetriever:** vector + lexical fusion; tunable weights.
* **Rerankers:** CrossEncoder baseline; optional LLM‑Reranker for hard queries.
* **Curator:** MMR/diversity, per‑doc caps, near‑dup pruning, budgeted context packing.
* **FusionRetriever (multimodal):** combine text/table/code/image retrievers with score normalization and modality quotas.

---

## 8) Agentic RAG (first‑class)

* **Planners:**

  * *ReActPlanner:* Thought→Action→Observation; uses `search()` as a tool.
  * *IRCoTPlanner:* interleave chain‑of‑thought with targeted retrieval per step.
  * *PlanDAGPlanner:* DAG of sub‑queries; parallel retrieval; structured assembly.
* **Active retrieval:** *ActiveGenerator (FLARE)* triggers lookups on low‑confidence spans; bounded by policies.
* **Routing & tools:** *RetrievalGate* (closed‑book vs retrieval; choose vector/lexical/web/graph). *Router* for domain/cost/metadata routing.
* **Memory & reflection:** Short‑term (hops), episodic (critiques), long‑term (verified facts). Optional self‑critique before finalize.
* **Termination:** max hops, no‑gain rounds, confidence/coverage thresholds, cost/token budgets, explicit FINISH.
* **Graph‑aware tool (optional):** build/query a knowledge graph; treat as a retrieval tool alongside text/table/image.

---

## 9) Cache RAG

* **Retrieval cache:** semantic key → curated chunk IDs + scores; partial‑hit merge with fresh search.
* **Synthesis cache:** query + pipeline hash → final answer + citations.
* **Freshness:** TTL + `data_version` on re‑ingest/re‑embed; per‑tenant scoping; negative caching. Cache keys encode `(store, collection, namespace, data_version, filters)`.

---

## 10) Generation, Grounding, Safety

* **LLMGenerator:** answer styles (concise, extractive, JSON, chain‑of‑evidence); citations policy.
* **ExtractiveReader:** non‑LLM span extractor for LLM‑free answers.
* **Guardrails:** Grounding validator (repair/refuse unsupported claims), PII redaction, policy checks; optional LLM‑judge.
* **Multimodal answers:** vision‑capable generators can consume images/tables alongside text.

---

## 11) Evaluation & Observability

* **IR metrics:** recall\@k, precision\@k, nDCG/MRR, diversity.
* **RAG grounding:** citation correctness, support/coverage, hallucination rate.
* **Agentic metrics:** hops, new‑evidence ratio, termination cause, per‑hop latency/cost, cache hit ratio.
* **A/B harness:** compare pipelines/configs; lifts/regressions; budget impact.
* **Tracing:** spans for stage1/2, rerank, MMR picks, planner actions, active lookups, cache events; exportable “evidence pack”.

---

## 12) Adapters for Popular Providers (kept separate, optional)

> SuperRAG core stays provider‑agnostic. Popular integrations ship as optional modules/packages implementing base interfaces. Names below are placeholders, not endorsements.

**Chat/LLM adapters (examples):**
`contrib.llm.ProviderAChatModel`, `ProviderBChatModel`, `ProviderCChatModel`, `LocalRuntimeChatModel`

**Embedding adapters (examples):**
`contrib.embed.ProviderAEmbedding`, `ProviderBEmbedding`, `LocalEmbedding`

**Vector/Sparse store drivers (examples):**
`contrib.index.VectorDB_A_Index`, `VectorDB_B_Index`, `VectorDB_C_Index`, `LexicalStore_A_Index`

**Rerankers/Judges (examples):**
`contrib.rank.ProviderA_LLMReranker`, `contrib.judge.ProviderBJudge`

**Install patterns:**

* Core: `pip install superrag`
* With common adapters: `pip install superrag[popular]`
* With a specific vendor: `pip install superrag[vendora]`

---

## 13) Developer Experience (DX)

* **Compose classes, not servers:** instantiate components; wire into pipelines; run in‑process.
* **Registry & entry‑points:** register/resolve components by name; third‑party plugins discoverable.
* **Profiles (pre‑wired graphs):** `standard`, `hybrid`, `doc_chunk_2stage`, `high_accuracy`, `cache_first`, `agentic`, `multimodal`, `compliance` — override any component/param.
* **LLM injection is explicit:** only Generator/Planner/LLM‑Reranker/Judge/ActiveGenerator accept `BaseChatModel`.
* **Interop:** adapters expose retrievers (LLM‑free) and ask/agent pipelines as retriever/tool interfaces in other ecosystems.
* **Testing:** contract tests for custom subclasses; golden sets; fixture corpora; trace pretty‑printer; A/B diff viewer.
* **CLI (optional):** ingest/reindex/eval wrappers calling the same classes.

---

## 14) Performance & Scale

* Parallel vector+lexical; batched embeddings/reranks; early‑cut rerank; per‑doc caps; diversity‑aware packing.
* ANN presets (low‑latency vs high‑recall); PQ/OPQ compression; index warmups; streaming ingestion with change detection.
* Namespaced sharding; store‑specific knobs (graph degree/ef/nprobe/alpha/quantization/consistency/region).
* Resilience: retries with jitter, partial‑batch recovery, circuit breakers; optional local fallback.

---

## 15) Ready‑Made Recipes (what teams build)

* **LLM‑free high‑precision retriever:** Hybrid + TwoStage + CrossEncoder + MMR + retrieval cache (local or managed store).
* **Cache‑first QA:** add LLMGenerator + synthesis cache + grounding guardrail.
* **High‑accuracy:** add HyDE/Multi‑Query planner, multi‑vector index, optional LLM‑rerank.
* **Agentic research assistant:** ReAct/IRCoT + ActiveGenerator + RetrievalGate + memory + strict termination.
* **Graph‑aware analyst:** PlanDAG + GraphTool + FusionRetriever (tables+text+images).
* **Compliance bot:** metadata routing, tenant filters, PII redaction, evidence packs, audit traces.

---

## 16) Roadmap (ship order)

1. Retrieval core: chunking suite, hybrid + two‑stage, cross‑encoder rerank, MMR, retrieval cache, tracing, adapters.
2. Answering & safety: LLMGenerator, citations, synthesis cache, grounding guardrail.
3. Agentic: ReAct/IRCoT planners, termination policies, ActiveGenerator (FLARE), RetrievalGate, memory.
4. Graph‑aware: KG tool + graph retriever; Plan‑DAG with parallel sub‑queries.
5. Eval UI & auto‑tune: agentic metrics dashboard; auto‑tuner for chunk size/overlap/rerank profile.
6. Distilled rerankers/judges: cheaper production variants.

---

### Bottom Line

SuperRAG is a **provider‑agnostic component library**: subclassable bases, swappable parts, **LLM‑optional** retrieval core, **agentic** modules you snap in, and **adapter packs** for popular LLMs, embedders, and vector databases. Start simple, scale to cutting‑edge, plug into any stack—without changing your mental model or running a service.
