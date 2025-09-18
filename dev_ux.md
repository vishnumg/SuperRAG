# SuperRAG — Usage with OpenAI LLMs & OpenAI Embeddings

> End‑to‑end examples that wire SuperRAG to **OpenAI chat models** (e.g., `gpt-4o`, `gpt-4o-mini`) and **OpenAI embeddings** (`text-embedding-3-*`). Retrieval remains **LLM‑free**; LLMs are used only where you inject them (generator/planner/reranker/judge).

---

## 0) Setup (shared imports)

```python
import os
# os.environ["OPENAI_API_KEY"] = "..."  # set in your environment

from superrag.ingestion import FSLoader
from superrag.chunking import SemanticChunker, StructuredChunker
from superrag.context import PrefixContextualizer
from superrag.embeddings import OpenAIEmbedding
from superrag.indexing import FaissIndex, BM25Index, PineconeIndex, QdrantIndex, WeaviateIndex
from superrag.retrieval import HybridRetriever, DocRetriever, ChunkRetriever, TwoStageRetriever, FusionRetriever
from superrag.rerank import CrossEncoderReranker, LLMReranker
from superrag.curate import DefaultCurator
from superrag.pipeline import SearchPipeline, AskPipeline, AgenticPipeline
from superrag.cache import RedisCache
from superrag.guardrails import GroundingGuardrail, PIIRedactor
from superrag.eval import Evaluator, AgenticEvaluator
from superrag.trace import SimpleTracer
from superrag.llm import OpenAIChatModel  # SuperRAG's BaseChatModel adapter
```

---

## 1) LLM‑free high‑precision search (OpenAI embeddings + FAISS)

```python
# Ingest
docs = FSLoader(paths=["./docs"]).load()
chunks = SemanticChunker(max_tokens=400, overlap_pct=0.15).split_many(docs)
chunks = PrefixContextualizer().enrich_many(chunks, docs)

# Embed with OpenAI
emb = OpenAIEmbedding(model="text-embedding-3-large")  # 3072‑dim; or use "text-embedding-3-small" (1536)
vecs = emb.embed_text([c.text for c in chunks])

# Index (dense + sparse)
faiss = FaissIndex(dim=emb.dim, metric="cosine").add(chunks, vecs)
bm25  = BM25Index().add(chunks)

# Compose LLM‑free pipeline
retriever = HybridRetriever(dense_index=faiss, sparse_index=bm25, stage1_k=24, stage2_k=64)
reranker  = CrossEncoderReranker(model="bge-reranker-large", top_k=8)
curator   = DefaultCurator(mmr_lambda=0.7, max_per_doc=2)
search    = SearchPipeline(retriever, reranker, curator, tracer=SimpleTracer())

hits = search.search("OAuth client secret rotation", k=12)  # no LLM used
```

---

## 2) Swap to managed vector DBs (still OpenAI embeddings)

```python
# Pinecone (native hybrid supported in many setups)
pine = PineconeIndex.from_uri("pinecone://proj:index@us-east-1?namespace=acme&metric=cosine")
pine.upsert(chunks, vecs)
retriever = HybridRetriever(dense_index=pine, sparse_index=None, hybrid_mode="native", stage1_k=24, stage2_k=64)

# Qdrant (logical hybrid with BM25)
qdr = QdrantIndex(collection="superrag", url="http://localhost:6333", metric="cosine").add(chunks, vecs)
retriever = HybridRetriever(dense_index=qdr, sparse_index=bm25, hybrid_mode="logical")

# Weaviate (native BM25+vector)
weav = WeaviateIndex(class_name="SuperRAG", url="http://localhost:8080").add(chunks, vecs)
retriever = HybridRetriever(dense_index=weav, sparse_index=None, hybrid_mode="native")

search = SearchPipeline(retriever, reranker, curator)
```

---

## 3) Two‑stage doc→chunk retrieval + Cross‑Encoder + MMR

```python
doc_ret   = DocRetriever(index=faiss, k=24)  # coarse via doc summary vectors
chunk_ret = ChunkRetriever(index=faiss, within_top_docs=doc_ret, k=64)
retriever = TwoStageRetriever(doc_ret, chunk_ret)
reranker  = CrossEncoderReranker(model="bge-reranker-large", top_k=10)
curator   = DefaultCurator(mmr_lambda=0.65, max_per_doc=2)
search    = SearchPipeline(retriever, reranker, curator)
```

---

## 4) Cache RAG — retrieval cache (no LLM) and synthesis cache

```python
search.enable_retrieval_cache(RedisCache(url="redis://localhost:6379"), ttl_seconds=86400)
# When using AskPipeline later, you can enable synthesis cache there as well.
```

---

## 5) AskPipeline with OpenAI LLM + strict citations

```python
# LLMs are explicit; retrieval remains LLM‑free
planner_llm = OpenAIChatModel(model="gpt-4o-mini")   # cheap model for rewrites/planning
gen_llm     = OpenAIChatModel(model="gpt-4o")        # main generator

ask = AskPipeline(
  search_pipeline=search,
  generator_model=gen_llm,  # BaseChatModel
  guardrails=[GroundingGuardrail(refuse_if_unsupported=True)]
)

answer = ask.ask("Summarize the SSO migration plan and key deadlines.")
print(answer.text)
print(answer.citations)
```

---

## 6) High‑accuracy retrieval: HyDE + Multi‑Query + optional LLM‑rerank

```python
from superrag.planning import HyDEPlanner, MultiQueryPlanner

planner = MultiQueryPlanner(n=3, rewriter=planner_llm)  # uses gpt‑4o‑mini only for rewrites
search.attach_planner(planner)

# Optional: LLM‑based reranker for hardest queries
llm_rank = LLMReranker(model=planner_llm, top_k=10)
search.set_reranker(llm_rank)  # remove to fall back to CrossEncoder

ans = ask.ask("Explain why incident TS-920 happened after mTLS rollout.")
```

---

## 7) Agentic ReAct multi‑hop with termination + memory

```python
from superrag.planning import ReActPlanner, TerminationPolicy
from superrag.memory import ShortTermMemory

react = ReActPlanner(llm=planner_llm, tools={"search": search.search})
stop  = TerminationPolicy(max_hops=4, no_new_evidence_rounds=1, max_cost_usd=0.10)

agent = AgenticPipeline(planner=react, search=search, generator=gen_llm,
                        memory=ShortTermMemory(window=6), termination=stop)

ans, trace = agent.run("Trace the chain of events that led to the auth rollback on March 12.")
trace.pretty_print()
```

---

## 8) FLARE‑style active retrieval during generation (OpenAI LLM)

```python
from superrag.generation import ActiveGenerator

active_gen = ActiveGenerator(
  llm=gen_llm,
  lookup=search.search,   # the retrieval tool to call mid‑generation
  strategy="flare",
  max_lookups=3
)

agent = AgenticPipeline(planner=None, search=search, generator=active_gen)
ans = agent.run("Draft an RCA and verify each claim against logs and runbooks.")
```

---

## 9) Multimodal retrieval & vision answers (gpt‑4o)

```python
from superrag.chunking import TableChunker, ImageCaptionChunker, CodeChunker
from superrag.embeddings import OpenAIEmbedding  # use text embeddings for captions/tables

# Build additional modality chunks
table_chunks = TableChunker().split_many(docs)
code_chunks  = CodeChunker().split_many(docs)
img_chunks   = ImageCaptionChunker().split_many(docs)  # captions become text for retrieval

# Reuse OpenAI embeddings for text/captions; index as usual
cap_vecs = emb.embed_text([c.text for c in img_chunks])
img_idx  = FaissIndex(dim=emb.dim).add(img_chunks, cap_vecs)

fusion = FusionRetriever(
  retrievers={
    "text": search.retriever,
    "table": ChunkRetriever(index=faiss, k=32),
    "image": ChunkRetriever(index=img_idx, k=24),
  },
  normalizer="zscore",
  balance={"text":0.6, "table":0.25, "image":0.15}
)

search_mm = SearchPipeline(retriever=fusion, reranker=reranker, curator=curator)
vision_llm = OpenAIChatModel(model="gpt-4o")  # vision‑capable
ask_mm = AskPipeline(search_mm, generator_model=vision_llm)

ans = ask_mm.ask("Show Q4 revenue by region and explain using the product diagram.")
```

---

## 10) Compliance / Governance mode

```python
ask.set_guardrails([
  GroundingGuardrail(refuse_if_unsupported=True),
  PIIRedactor(),
])
# Enforce metadata/tenant filters at retrieval time
answer = ask.ask(
  "What is our policy on exporting user data to vendors?",
  query_metadata={"tenant":"acme","filters":{"doctype":"policy"}}
)
```

---

## 11) Evaluation & A/B (OpenAI embeddings everywhere)

```python
calibration = [
  {"q":"Rotate OAuth client secrets?","gold_docs":["security_runbook.md"]},
  {"q":"PCI network segmentation controls","gold_docs":["pci_controls.pdf"]},
]

report_a = Evaluator(search).run(calibration, metrics=["recall@10","precision@5","latency"])  
report_b = Evaluator(search_mm).run(calibration, metrics=["recall@10","precision@5","nDCG"])
print(report_b.delta(report_a))
```

---

## 12) Per‑request overrides & budgets (RAG‑native BaseChatModel)

```python
summary = ask.ask(
  "Create an exec summary with citations",
  llm_overrides={"generator": OpenAIChatModel(model="gpt-4.1-mini")},
  llm_policy={"generator": {"max_tokens": 2000, "max_calls": 1, "max_cost_usd": 0.50}}
)

# Force LLM‑free retrieval even if planners exist
hits = search.search("TLS cipher suites audit", llm_off=True)
```

---

## 13) Tenancy, filters, and incremental re‑ingestion

```python
# Scoped retrieval by tenant/env/type
hits = search.search(
  "GDPR retention settings",
  query_metadata={"tenant":"acme","env":"prod","filters":{"doctype":"policy"}}
)

# Re‑embed only changed files; data_version bump auto‑invalidates caches
# loader = FSLoader(paths=["./docs"]).watch_for_changes()
# on change → rechunk, re‑embed with OpenAIEmbedding, index.upsert(...), cache.bump_data_version()
```

---

## 14) LangChain / AutoGen adapters (OpenAI under the hood)

```python
from superrag.adapters.langchain import as_retriever, as_runnable
lc_retriever = as_retriever(search)        # LLM‑free retriever
lc_chain     = as_runnable(ask)            # Runnable using your OpenAI generator

from superrag.adapters.autogen import as_tool
tool_search = as_tool(search, name="search_superrag")
tool_ask    = as_tool(ask,    name="ask_superrag")
```

---

### Notes

* **Retrieval is always LLM‑free.** Using OpenAI **embeddings** does not invoke a chat model.
* **LLM calls are explicit** via `OpenAIChatModel` and only in components you wire (generator/planner/reranker/judge).
* **Managed DBs**: flip between FAISS/Pinecone/Qdrant/Weaviate without changing anything above the index.
* Prefer `text-embedding-3-large` for highest recall (3072‑d); use `text-embedding-3-small` to cut cost/size (1536‑d).
