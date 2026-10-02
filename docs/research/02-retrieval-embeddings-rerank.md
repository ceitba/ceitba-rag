# 02 — Retrieval design: embeddings, hybrid search, reranking, exercise linking, dedup

Status: research draft, 2026-10-02. Prices are list prices found on the dates cited; re-check before committing.

## Context

- Goal: for a Spanish natural-language query, return the top-5 links to student-note pages, with more available on request. The system never generates solutions.
- Query shapes:
  - (a) ID-style: "Ej 23 Guía 2 Probabilidad", "1P 2024 2C ej 3".
  - (b) statement or plain-text math: "integral de x^2 sen x".
  - (c) topic: "teorema central del límite".
- Corpus: about 1.5M pages, one record per page. A VLM transcribes handwritten math into markdown+LaTeX. Each page has metadata: subject code (CEITBA API), doc type (guide/theory/1P/2P/1R/1F/2F/lab/TP), year, term, guide number, exercise numbers, owner, source.
- Stack: self-hosted Postgres+pgvector on a VPS, Go query API, Python workers. Load is about 10k queries/day (≈300k/month) with exam-season bursts. Cloud APIs are preferred over self-hosted models.
- Working assumptions, to verify on a sample:
  - About 700 tokens per page (LaTeX tokenizes heavily), so the corpus is ≈1.05B tokens.
  - A query is about 40 tokens including the instruction.
  - Rerank candidates are truncated to about 400 tokens.

## Embeddings

Benchmark scores are MMTEB "MTEB (Multilingual) Mean (Task)" plus its Retrieval column where published. I found no Spanish-only retrieval score published consistently across these models. A Spanish+math eval on our own data is mandatory (see Open questions).

| Model | MMTEB mean / retrieval | Dims (Matryoshka) | Context | Price / 1M tokens | Batch | Notes |
|---|---|---|---|---|---|---|
| Qwen3-Embedding-8B | 70.58 / 70.88 | 32–4096 (any) | 32k | ≈$0.01–0.04 hosted ([Vercel](https://vercel.com/ai-gateway/models/qwen3-embedding-8b), [OpenRouter providers](https://openrouter.us.helicone.ai/qwen/qwen3-embedding-8b/providers)); Alibaba `text-embedding-v4` ≈$0.07 ([cloudprice](https://cloudprice.net/models/alibaba-text-4-embedding)) | provider-dependent | Apache-2.0 open weights, instruction-aware (+1–5% with query instruction) ([HF card](https://huggingface.co/Qwen/Qwen3-Embedding-8B)). The 4B (69.45/69.60) and 0.6B (64.33/64.64) variants can be self-hosted on one GPU. |
| Gemini Embedding 2 (GA) | 69.9 (retrieval n/a) | 128–3072 | 8,192 | $0.20 text; $0.00012/image | $0.10 text, $0.00006/image | Text, image, PDF and audio share one embedding space ([DeepMind](https://deepmind.google/models/gemini/embedding/), [pricing](https://ai.google.dev/gemini-api/docs/pricing)). `gemini-embedding-001` (68.4) is still listed at $0.15 ([Google blog](https://developers.googleblog.com/gemini-embedding-available-gemini-api/)). |
| Voyage 4 family (`voyage-4-large` / `-4` / `-4-lite`) | not on MMTEB (vendor RTEB claims) | 2048/1024/512/256; float/int8/binary | 32k | $0.12 / $0.06 / $0.02; first 200M tokens free | −33% (no free tokens) | One embedding space across sizes, so you can index with `large` and query with `lite` ([pricing](https://docs.voyageai.com/docs/pricing), [FAQ](https://docs.voyageai.com/docs/faq), [Azure card](https://ai.azure.com/catalog/models/voyage-4-embedding-model)). |
| `voyage-context-4` | — | as above | 32k | $0.12 | −33% | Each chunk (= our page) is embedded with whole-document context ([MongoDB](https://www.mongodb.com/docs/voyageai/models/contextualized-chunk-embeddings/)). Relevant for continuation pages that lack the exercise header. |
| Cohere Embed 5 Pro / Fast | not on MMTEB; ViDoRe v3 85.8 / 84.5 | 2048…256; float/int8/binary | 128k | $0.12 / $0.08 text; $0.40 image | — | Pro and Fast share one space; text+image input ([Cohere](https://cohere.com/blog/embed-5)). Embed v4 is ≈$0.12 ([eesel](https://www.eesel.ai/blog/cohere-ai-pricing)). |
| OpenAI `text-embedding-3-large` / `-small` | 58.93 / 59.27 (large) | 3072 / 1536, truncatable | 8k | $0.13 / $0.02 | −50% | Clearly behind on multilingual ([pricing](https://costgoat.com/pricing/openai-embeddings); scores from the [Qwen card](https://huggingface.co/Qwen/Qwen3-Embedding-8B)). |
| jina-embeddings-v5-text-small / v4 | n/a here | 1024 (32–1024) / 2048 | 32k | ≈$0.05 (shared wallet) | — | Weights are CC-BY-NC-4.0, so self-hosting needs a commercial license from Elastic ([Jina llms.txt](https://jina.ai/models/llms.txt), [markaicode](https://markaicode.com/pricing/jina-ai-pricing/)). v4 also has a multimodal / multi-vector mode. |
| BGE-M3 | 59.56 / 54.60 | 1024 + sparse + ColBERT | 8k | self-host only | — | MIT. Weak dense retrieval by 2026 standards; its sparse head is the main reason to consider it ([paper](https://arxiv.org/abs/2402.03216)). |

Takeaways:
- Every option costs ≤$210 to embed the full corpus once (see Cost), so re-embedding is cheap and lock-in is low. Choose on eval quality and API reliability.
- An open-weight model such as Qwen3 removes the risk of a provider deprecating the model. Query vectors must always come from the same model as the index, and with open weights we can self-host it if needed.
- Math mismatch: users type "x^2 sen x" while pages contain `x^{2}\sin x`. Embeddings bridge part of this gap, but not reliably. At index time, also store a linearized plain-text version of each page (LaTeX → text, `\sin`→`sen`, `\int`→`integral`). Embed `plain_text` (optionally prefixed with metadata such as "Probabilidad · Guía 2 · Ej 23"), and also index it for BM25. Apply the same normalizer (sen/sin, ^, sqrt/raíz) to queries.

## Multimodal alternative

- **ColPali / ColQwen (late interaction):**
  - Each page yields hundreds to thousands of patch vectors ([Visual RAG Toolkit](https://arxiv.org/abs/2602.12510)). At ~1k × 128-d fp16 per page, 1.5M pages is ≈380 GB.
  - pgvector has no MaxSim index, and there is no mainstream cloud API, so it needs self-hosted GPUs.
  - Pooling down to dozens of vectors per page still leaves ~10+ GB plus custom scoring.
  - Rejected as the primary path. The best late-interaction models (e.g. the Qwen3-VL-based 8B, ViDoRe v3 nDCG@10 63.42, [arXiv 2602.03992](https://arxiv.org/html/2602.03992v1)) are research-grade at this scale.
- **Single-vector page-image APIs** fit our design as an extra RRF leg:
  - Gemini Embedding 2 image: $0.00012/page, $0.00006 batch, so ≈$90–180 for the corpus. It shares a space with its text queries.
  - voyage-multimodal-3.5: $0.60 per billion pixels, so a 1 MP page costs $0.0006. That is ≈$810 after 150B free pixels ([pricing](https://docs.voyageai.com/docs/pricing)).
  - Cohere Embed 5: images at $0.40/1M tokens.
  - Qwen3-VL-Embedding 2B/8B: open, self-host only ([arXiv 2601.04720](https://arxiv.org/abs/2601.04720)).
  - jina v5-omni: non-commercial license.
- Verdict:
  - The VLM transcription already captures the content, so text-from-OCR stays primary.
  - Image embeddings are worth an A/B on ~2k pages. Expected value: diagrams and graphs, pages where the transcription is poor, and a dedup signal.
  - If recall@50 improves, add rows with `modality='image'` to `page_embeddings` and a third RRF leg.
  - Text-to-image retrieval over Spanish handwritten math is unproven in public benchmarks.

## Hybrid search in Postgres

**pgvector (current v0.8.7)** ([README](https://github.com/pgvector/pgvector)):
- Index dimension limits: `vector` HNSW up to 2,000 dims, `halfvec` up to 4,000, `bit` up to 64,000. 3072-d therefore requires halfvec or binary quantization.
- Use `halfvec` storage and indexes. Index cost is half of float32, with negligible recall loss.
- Binary quantization is an expression index `binary_quantize(embedding)::bit(n)` plus a re-score by the full vector. Not needed at 1.5M × 1024 (see Sizing). Keep it as a scale fallback; prefer native int8/binary output if we choose Voyage or Cohere.
- Filtered search: 0.8 added iterative index scans (`hnsw.iterative_scan = relaxed_order`, cap via `hnsw.max_scan_tuples`) and better planner costing ([PG news](https://www.postgresql.org/about/news/pgvector-080-released-2952/)). This means `WHERE subject_code = $1 AND doc_type = ANY($2)` no longer returns too few rows.
  - Put the filter columns directly on the embeddings table, because filters behind a join aren't pushed into the index scan.
  - Very selective filters (one guide, one year) fall back to exact scan over a B-tree. That is cheap at ≤50k rows.

**Lexical (BM25) options:**

| Option | Spanish | Ranking | Notes |
|---|---|---|---|
| Native FTS: custom config `es_unaccent` (copy of `spanish`, `unaccent` mapped before `spanish_stem`), stored generated `tsvector`, GIN | Snowball stemmer + unaccent | `ts_rank_cd` (no IDF, no top-k index) | Zero new extensions. Ranking must score every match, so common terms ("integral") get slow across 1.5M rows. Acceptable with tight metadata filters. |
| ParadeDB `pg_search` (Tantivy) | Snowball `stemmer=spanish` ([docs](https://www.paradedb.com/docs/documentation/token-filters/stemming)) | True BM25 with top-k, multiple fields, separate search tokenizer ([docs](https://www.paradedb.com/docs/documentation/tokenizers/search-tokenizer)) | PG 15+, plain extension with no fork ([intro](https://docs.paradedb.com/welcome/introduction)). Check the license (AGPL) before distribution. Recommended. |
| VectorChord-bm25 + `pg_tokenizer` | LLM tokenizers (gemma2b, llmlingua2) or custom ([pgEdge docs](https://docs.pgedge.com/vchord-bm25/development/tokenizer-options/)) | BM25, Block-WeakAnd ([repo](https://github.com/tensorchord/VectorChord-bm25)) | Spanish stemming is less turnkey. The sibling VectorChord (RaBitQ IVF) builds far faster than pgvector HNSW ([blog](https://blog.vectorchord.ai/vectorchord-10-developer-first-vector-search-on-postgres-100x-faster-indexing-than-pgvector)); keep it in mind if HNSW build time hurts. |

**Index two lexical fields:**
- `plain_text` with the Spanish stemmer.
- `math_norm`, built with a whitespace/regex tokenizer and no stemming, so that tokens like `x^2`, `sen`, `e^x` and `\int` survive.

**Fusion:**
- Each leg returns 100 candidates under identical metadata filters.
- Combine them with RRF: `score = Σ 1/(60 + rank_i)`.
- Collapse by `dup_group_id`, then rerank the top 50.
- RRF needs no score calibration and is SQL-only (`FULL OUTER JOIN` of two ranked CTEs) ([ParadeDB manual](https://www.paradedb.com/blog/hybrid-search-in-postgresql-the-missing-manual)).

## Reranking

Cost for reranking the top-50 with 400-token docs and a 30-token query works out to ≈21.5k tokens per query for token-billed APIs.

| Reranker | Billing | Est. $ / 1k queries (top-50) | Notes |
|---|---|---|---|
| Voyage `rerank-3` / `rerank-3-lite` | $0.05 / $0.02 per 1M tokens; billed tokens = query × docs + Σ doc tokens; 200M free on 2.5/2 generations | $1.08 / $0.43 | 32k context; current Voyage recommendation ([docs](https://docs.voyageai.com/docs/reranker), [pricing](https://docs.voyageai.com/docs/pricing)). `rerank-2.5` added instruction-following ([MongoDB](https://www.mongodb.com/company/blog/product-release-announcements/rerank-2-5-and-rerank-2-5-lite-instruction-following-rerankers)). |
| Cohere Rerank 4 Pro / Fast | $2.50 / $2.00 per 1k searches; 1 unit = query + ≤100 docs, docs >~500 tokens (incl. query) split into chunks ([Pinecone](https://docs.pinecone.io/models/cohere-rerank-4-fast), [puter](https://developer.puter.com/tutorials/cohere-api-pricing/)) | $2.50 / $2.00 | Truncate docs to ≤450 tokens to stay at 1 unit; multilingual, 32k ([changelog](https://docs.cohere.com/changelog/rerank-v4.0)). |
| Jina reranker v3 / v3.5 | ≈$0.05 / 1M tokens wallet | ≈$1.08 (token accounting unverified) | 0.6B listwise; v3.5 reports 63.20 BEIR nDCG@10 ([Jina](https://jina.ai/news/jina-reranker-v3-5-faster-listwise-reranking-hybrid-attention-self-distillation/)). Weights non-commercial. |
| Qwen3-Reranker (0.6B/4B/8B) | DashScope `qwen3-rerank` ≈$0.10 / 1M ([LiteLLM PR](https://github.com/BerriAI/litellm/pull/29163)); 8B from ≈$0.05 / 1M ([cloudprice](https://cloudprice.net/models/alibaba-qwen3-reranker-8b)) | $2.15 / $1.08 | Apache-2.0, instruction-aware, 32k. |
| bge-reranker-v2-m3 | self-host only | GPU ≈$0.89/h (A100 on DeepInfra, [morphllm](https://www.morphllm.com/deepinfra-pricing)), ≈$650/mo 24/7 | Too slow on CPU for 50×400 tokens at interactive latency. Not justified at our volume. |

Rerank is the dominant per-query cost. Mitigations:
- Skip rerank when the parser resolved an exact `exercise_id`.
- Cache by normalized query plus filters. Exam-season queries repeat heavily.
- Dedup-collapse before reranking.
- Rerank 30 candidates instead of 50 when the leg scores agree.
- Rerank once and serve "more results" from the cached ranked list.

## Exercise linking

Goal: "Ej 23 Guía 2 Probabilidad" and the pasted statement text should both resolve to the same `exercise_id`, and therefore return the same pages.

1. **Canonical exercises (offline).**
   - Parse the official guide and exam documents (`documents.is_official`, e.g. `Practica/G2 - Calculo De Probabilidades.pdf`, `Exámenes Viejos/*`). An LLM converts each into structured JSON: `{label: "23", sub: "b", statement_md}`.
   - Store each exercise with `(subject_code, set_kind, guide_number | exam (year, term, type), edition_year, label)`.
   - Embed `statement_plain` with the same model as pages. The table stays small (≈10⁴–10⁵ rows), so a tiny HNSW index is enough.
   - When guides are renumbered across editions, link them by statement similarity through `canonical_exercise_id`.
2. **Link student pages (offline worker).**
   - Candidate labels come from the transcription (regex: `Ej(ercicio)?\.?\s*(\d+)([a-z])?`, `^\s*(\d+)\s*[).]`) plus document metadata (subject, guide).
   - **Label + verify:** if a page has a label and `cos(page_or_segment, exercise.statement) ≥ τ_low`, link it with method `label+sim`.
   - **Similarity only:** with no label, run kNN over `exercises` within the subject. Link if `cos ≥ τ_high` and the margin over the runner-up is `≥ δ`.
   - **Ambiguous band:** an LLM adjudicates using the page plus the top-3 statements, for about $0.0003 per call.
   - **Continuation pages** (no header) inherit the previous page's exercise with role `solution` and decayed confidence. `voyage-context-4` can also help here.
   - Calibrate `τ`/`δ` on ~500 hand-labelled pairs.
3. **Query time.**
   - The parser outputs `{subject_code?, doc_type?, year?, term?, guide_number?, exercise_label?, statement_text?, topic_text}`.
   - Regex and lookup first: subject aliases via `pg_trgm` on `subjects.name_norm`, plus `1P|2P|1R|final`, `1C|2C`, `gu[ií]a \d+`, `ej(ercicio)? \d+`.
   - A small LLM with JSON output runs only when the regex result is incomplete, or when free-text math is present.
   - Resolved `exercise_id` → `page_exercise_links` ordered by confidence and quality, collapsed by dup group, with no rerank needed.
   - `statement_text` → kNN on `exercises` first. If the top hit clears `τ_high`, inject its linked pages at the top. Otherwise use the normal hybrid search.

## Dedup

Use three signals. Never merge pages on embedding similarity alone: two students solving the same exercise score cosine ≥0.9 and are *not* duplicates.

| Signal | Method | Candidate threshold (calibrate) |
|---|---|---|
| Exact | SHA-256 of the file and of the normalized transcription | equal |
| Image | 64-bit pHash/dHash per page image (`imagehash`), stored as `bit(64)` and indexed with pgvector HNSW `bit_hamming_ops` | Hamming ≤ 8 |
| Text | MinHash (128 perms, char 5-shingles on lowercased, unaccented, whitespace-collapsed transcription; LSH 32×4 via `datasketch`) | Jaccard ≥ 0.8 |
| Embedding | cosine on the page embedding | ≥ 0.97, only as a tiebreaker |

Merge rules:
- Create an edge if (pHash ≤ 8 AND Jaccard ≥ 0.6) OR Jaccard ≥ 0.9 OR exact match.
- Build groups by union-find into `dup_groups`. The canonical page has the highest quality score: resolution, transcription confidence, official/verified source, most recent.
- Promote to document-level dedup when ≥80% of pages match.

Use of groups:
- Index only canonical pages in the HNSW and BM25 indexes (partial indexes), which also shrinks them.
- The UI shows "N copies". Owner attribution stays per page.

## Sizing (1.5M vectors)

Estimates. Raw size = 4d+8 bytes (vector) or 2d+8 (halfvec). HNSW `m=16` adds ≈300 B/vector of graph plus page overhead (×≈1.15).

| Dims | float32 raw | float32 HNSW | halfvec raw | halfvec HNSW | binary (BQ) HNSW |
|---|---|---|---|---|---|
| 1024 | 6.2 GB | ≈7.6 GB | 3.1 GB | ≈4.0 GB | ≈0.7 GB (+ re-score from heap) |
| 1536 | 9.2 GB | ≈11 GB | 4.6 GB | ≈5.8 GB | ≈0.8 GB |
| 2048 | 12.3 GB | not indexable (>2000) | 6.2 GB | ≈7.6 GB | ≈0.9 GB |
| 3072 | 18.4 GB | not indexable | 9.2 GB | ≈11 GB | ≈1.1 GB |

- The heap stores another copy of the raw vectors. Only the rows touched for re-scoring need to be hot.
- BM25/FTS indexes over ~1B tokens are likely 2–5 GB (unmeasured).
- **Recommendation: 1024-d halfvec, ≈4 GB index.**
  - Use a VPS with ≥32 GB RAM (16 GB is the minimum).
  - Set `shared_buffers` ≈ 8 GB so the HNSW index stays in shared buffers or the OS page cache.
  - Build with `maintenance_work_mem` ≈ 6 GB and 4–8 parallel maintenance workers.
  - `ef_search` 80–120 with iterative scan.
- 1536 or 2048 still fits on 32–64 GB if the eval shows a real gain. Avoid 3072.

## Cost

Volume: 300k queries/month.

**One-time corpus embedding (≈1.05B tokens):**

| Model | Cost |
|---|---|
| Qwen3-8B hosted | ≈$10–40 |
| voyage-4 | ≈$51 after free tier, ≈$42 batch |
| voyage-4-large / context-4 | ≈$126, ≈$84 batch |
| Cohere Embed 5 Pro | ≈$126 |
| Gemini Embedding 2 | ≈$210, $105 batch |
| OpenAI 3-large | ≈$137, $68 batch |

The optional image leg with Gemini Embedding 2 adds ≈$90 (batch).

**Per query:**

| Component | Assumption | $/query | $/month |
|---|---|---|---|
| Query embedding | 40 tokens, ≤$0.06/M (Qwen3 / voyage-4; Gemini $0.20) | ≤$0.0000024 (Gemini $0.000008) | ≤$1 (Gemini ≈$2.4) |
| Rerank top-50, Voyage `rerank-3` | 21.5k tokens | $0.00108 | $323 |
| Rerank top-50, Voyage `rerank-3-lite` | | $0.00043 | $129 |
| Rerank top-50, Cohere Rerank 4 Pro / Fast | 1 unit | $0.0025 / $0.0020 | $750 / $600 |
| Query-parsing LLM (Gemini 3.1 Flash-Lite, $0.25/$1.50 per 1M, [Google](https://blog.google/innovation-and-ai/technology/developers-tools/gemini-3-1-flash-lite/)) | 500 in / 60 out | $0.000215 | $65 at 100%; ≈$19 if only 30% hit the LLM fallback |

Monthly totals (excluding the VPS):

| Profile | Components | Total |
|---|---|---|
| Lean | Qwen3 query embedding + `rerank-3-lite` on 60% of queries (cache/ID bypass) + 30% LLM fallback | ≈$100 |
| Default | Qwen3 + `rerank-3` on 60% + 30% LLM | ≈$215 |
| Max | `rerank-3` on all queries + LLM on all | ≈$390 |
| Cohere Pro on all + LLM on all | | ≈$815 |

Bursts (≈5× daily average) are a rate-limit question, not a cost one. Confirm RPM/TPM tiers with the chosen vendors.

## Recommendation

**Stack:**
- **Embeddings:** `Qwen3-Embedding-8B` at 1024-d (MRL) via a hosted API, with an English task instruction on queries. Store as `halfvec(1024)`. The open weights let us self-host the same model later.
  - Run a bake-off before locking in: Gemini Embedding 2 @1024, and voyage-4-large (index) with voyage-4-lite (query) @1024.
  - Switch if either wins by more than ~2 points nDCG@10 on our eval.
- **Text to embed:** linearized `plain_text`, prefixed with metadata. Raw LaTeX stays in `transcription_md` for display and BM25.
- **Vector index:** pgvector 0.8.7 HNSW (`halfvec_cosine_ops`, m=16, ef_construction=128), partial on `is_canonical`, with iterative scans for filtered queries.
- **Lexical:** ParadeDB `pg_search` BM25 on `plain_text` (Spanish stemmer) and `math_norm`. If no extension is allowed, fall back to native FTS with `es_unaccent` + GIN.
- **Fusion:** RRF (k=60) over vector top-100 and BM25 top-100, optionally adding exercise-kNN results. Then dup-collapse, then Voyage `rerank-3` on the top-50 (or `-lite` in the lean profile). Return 5 and cache the ranked list for "more".
- **Parser:** regex and lookup first, with Gemini 3.1 Flash-Lite as a JSON fallback.
- **Exercise path:** short-circuits rerank when an exercise is resolved.
- **Ops:** the Go API is public, so add auth or rate limiting per client.

**Schema sketch:**

```sql
CREATE EXTENSION IF NOT EXISTS vector;     -- 0.8.7
CREATE EXTENSION IF NOT EXISTS unaccent;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS pg_search;  -- ParadeDB (optional)

CREATE TABLE subjects (                   -- synced from CEITBA API
  code text PRIMARY KEY,                  -- '93.24'
  name text NOT NULL,
  name_norm text NOT NULL                 -- unaccent(lower(name)) for pg_trgm alias matching
);
CREATE INDEX ON subjects USING gin (name_norm gin_trgm_ops);

CREATE TABLE sources (
  id bigserial PRIMARY KEY,
  kind text NOT NULL,                     -- 'drive','upload','ceitba_api',...
  uri text NOT NULL,
  owner text,
  fetched_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE documents (
  id bigserial PRIMARY KEY,
  source_id bigint NOT NULL REFERENCES sources,
  subject_code text REFERENCES subjects,
  doc_type text NOT NULL CHECK (doc_type IN ('guide','practice','theory','1P','2P','1R','2R','1F','2F','3F','lab','TP','other')),
  year smallint, term text CHECK (term IN ('1C','2C','V')),
  guide_number smallint,
  owner text,
  title text,
  is_official boolean NOT NULL DEFAULT false,
  file_sha256 bytea UNIQUE,
  page_count int
);
CREATE INDEX ON documents (subject_code, doc_type, year, term);

CREATE TABLE dup_groups (
  id bigserial PRIMARY KEY,
  canonical_page_id bigint,               -- FK added after pages
  member_count int NOT NULL,
  method text NOT NULL                    -- 'exact','phash+minhash','minhash'
);

CREATE TABLE pages (
  id bigserial PRIMARY KEY,
  document_id bigint NOT NULL REFERENCES documents ON DELETE CASCADE,
  page_no int NOT NULL,
  image_uri text NOT NULL,
  transcription_md text NOT NULL,         -- VLM markdown+LaTeX
  plain_text text NOT NULL,               -- LaTeX linearized to Spanish text
  math_norm text NOT NULL,                -- normalized math tokens
  exercise_refs text[] NOT NULL DEFAULT '{}',
  content_sha256 bytea NOT NULL,
  phash bit(64),
  quality real,
  dup_group_id bigint REFERENCES dup_groups,
  is_canonical boolean NOT NULL DEFAULT true,
  -- denormalized filters
  subject_code text, doc_type text, year smallint, term text, guide_number smallint,
  tsv tsvector GENERATED ALWAYS AS (to_tsvector('es_unaccent', plain_text)) STORED,  -- native-FTS fallback
  UNIQUE (document_id, page_no)
);
ALTER TABLE dup_groups ADD FOREIGN KEY (canonical_page_id) REFERENCES pages;
CREATE INDEX ON pages (subject_code, doc_type, year, term, guide_number) WHERE is_canonical;
CREATE INDEX ON pages USING gin (tsv) WHERE is_canonical;               -- fallback
CREATE INDEX ON pages USING hnsw (phash bit_hamming_ops);               -- dedup candidates
CREATE INDEX pages_bm25 ON pages USING bm25 (id, plain_text, math_norm) -- ParadeDB; configure
  WITH (key_field = 'id');                                              -- spanish stemmer / regex tokenizer per field

CREATE TABLE page_embeddings (
  page_id bigint NOT NULL REFERENCES pages ON DELETE CASCADE,
  model text NOT NULL,                    -- 'qwen3-emb-8b@1024'
  modality text NOT NULL DEFAULT 'text',  -- 'text' | 'image'
  segment_no smallint NOT NULL DEFAULT 0, -- 0 = whole page; >0 = per-exercise segment
  embedding halfvec(1024) NOT NULL,
  -- denormalized so iterative HNSW scans can filter without joins
  subject_code text, doc_type text, year smallint, term text, is_canonical boolean NOT NULL,
  PRIMARY KEY (page_id, model, modality, segment_no)
);
CREATE INDEX page_emb_hnsw ON page_embeddings
  USING hnsw (embedding halfvec_cosine_ops) WITH (m = 16, ef_construction = 128)
  WHERE model = 'qwen3-emb-8b@1024' AND modality = 'text' AND is_canonical;
CREATE INDEX ON page_embeddings (subject_code, doc_type, year) WHERE is_canonical;

CREATE TABLE exercises (
  id bigserial PRIMARY KEY,
  subject_code text NOT NULL REFERENCES subjects,
  set_kind text NOT NULL CHECK (set_kind IN ('guide','exam')),
  guide_number smallint,                  -- for guides
  exam_doc_type text, exam_year smallint, exam_term text,  -- for exams
  edition_year smallint,
  label text NOT NULL,                    -- '23', '23b'
  parent_id bigint REFERENCES exercises,  -- sub-items
  canonical_exercise_id bigint REFERENCES exercises,  -- cross-edition alias
  source_page_id bigint REFERENCES pages,
  statement_md text NOT NULL,
  statement_plain text NOT NULL,
  statement_embedding halfvec(1024) NOT NULL,
  UNIQUE NULLS NOT DISTINCT (subject_code, set_kind, guide_number, exam_doc_type, exam_year, exam_term, edition_year, label)
);
CREATE INDEX ON exercises USING hnsw (statement_embedding halfvec_cosine_ops);
CREATE INDEX ON exercises (subject_code, set_kind, guide_number, label);

CREATE TABLE page_exercise_links (
  page_id bigint NOT NULL REFERENCES pages ON DELETE CASCADE,
  exercise_id bigint NOT NULL REFERENCES exercises ON DELETE CASCADE,
  role text NOT NULL CHECK (role IN ('statement','solution','partial')),
  method text NOT NULL CHECK (method IN ('label','label+sim','sim','llm','continuation','manual')),
  confidence real NOT NULL,
  PRIMARY KEY (page_id, exercise_id)
);
CREATE INDEX ON page_exercise_links (exercise_id, confidence DESC);
```

**Query flow in Go:**
1. Normalize the query, then check the cache.
2. Parse with regex, falling back to the LLM.
3. If an exercise resolves → links → collapse → return.
4. Otherwise run three legs in parallel: exercise kNN, page kNN (filtered, iterative scan), and BM25 (filtered).
5. RRF → collapse by `dup_group_id` → rerank top-50 → top-5. Cache the ranked list.

## Open questions

- **Eval set.** Build 300–500 labelled queries across shapes (ID / statement / plain math / topic) and subjects. Metrics: recall@100 before rerank, then nDCG@5 and MRR. Nothing above replaces this. I found no published Spanish-only or Spanish-math retrieval numbers for these models.
- **Real tokens per page and per segment.** This drives the corpus cost and the rerank truncation strategy (~400 tokens, header-first).
- **Official guides and exams.** Do we have them for every subject and edition? Without them, exercise linking falls back to similarity-only clusters of student pages.
- **voyage-context-4.** Check its per-document token limit against our longest documents; it is useful for continuation pages.
- **Licensing.** ParadeDB `pg_search` (AGPL) and VectorChord need a license check; Jina weights are non-commercial.
- **Unverified billing and sizes.** Jina rerank token accounting, and Cohere Rerank 4 chunking at our doc lengths, should be confirmed on a test invoice. The HNSW and BM25 sizes above are estimates; measure on a 100k-page sample.
- **Privacy.** Should the `owner` field be exposed in results, and do owners need opt-out/takedown? This affects whether `pages` needs an `is_visible` flag in every index predicate.
- **VPS spec.** Confirm RAM and CPU. A ≥32 GB host is assumed, with no GPU.

## Sources

- Qwen3-Embedding-8B model card (MMTEB table, dims, instructions): https://huggingface.co/Qwen/Qwen3-Embedding-8B
- Qwen3 embedding hosted pricing: https://vercel.com/ai-gateway/models/qwen3-embedding-8b · https://openrouter.us.helicone.ai/qwen/qwen3-embedding-8b/providers · https://cloudprice.net/models/alibaba-text-4-embedding
- Gemini Embedding 2 specs and benchmarks: https://deepmind.google/models/gemini/embedding/ · pricing: https://ai.google.dev/gemini-api/docs/pricing · gemini-embedding-001: https://developers.googleblog.com/gemini-embedding-available-gemini-api/
- Voyage pricing (embeddings, multimodal, rerank, batch): https://docs.voyageai.com/docs/pricing · rerankers: https://docs.voyageai.com/docs/reranker · FAQ: https://docs.voyageai.com/docs/faq · voyage-4 dims/quantization: https://ai.azure.com/catalog/models/voyage-4-embedding-model · voyage-context-4: https://www.mongodb.com/docs/voyageai/models/contextualized-chunk-embeddings/ · rerank-2.5: https://www.mongodb.com/company/blog/product-release-announcements/rerank-2-5-and-rerank-2-5-lite-instruction-following-rerankers
- Cohere Embed 5: https://cohere.com/blog/embed-5 · Rerank 4: https://docs.cohere.com/changelog/rerank-v4.0 · rerank units: https://docs.pinecone.io/models/cohere-rerank-4-fast · https://developer.puter.com/tutorials/cohere-api-pricing/ · Embed v4: https://www.eesel.ai/blog/cohere-ai-pricing
- OpenAI embeddings pricing: https://costgoat.com/pricing/openai-embeddings
- Jina models and licensing: https://jina.ai/models/llms.txt · pricing: https://markaicode.com/pricing/jina-ai-pricing/ · reranker v3.5: https://jina.ai/news/jina-reranker-v3-5-faster-listwise-reranking-hybrid-attention-self-distillation/
- Qwen3 rerank pricing: https://github.com/BerriAI/litellm/pull/29163 · https://cloudprice.net/models/alibaba-qwen3-reranker-8b
- BGE-M3: https://arxiv.org/abs/2402.03216
- ColPali scaling: https://arxiv.org/abs/2602.12510 · late-interaction ViDoRe v3: https://arxiv.org/html/2602.03992v1 · Qwen3-VL-Embedding: https://arxiv.org/abs/2601.04720
- pgvector: https://github.com/pgvector/pgvector · 0.8.0 release: https://www.postgresql.org/about/news/pgvector-080-released-2952/
- ParadeDB: https://www.paradedb.com/docs/documentation/token-filters/stemming · https://www.paradedb.com/docs/documentation/tokenizers/search-tokenizer · https://docs.paradedb.com/welcome/introduction · https://www.paradedb.com/blog/hybrid-search-in-postgresql-the-missing-manual
- VectorChord: https://github.com/tensorchord/VectorChord-bm25 · https://docs.pgedge.com/vchord-bm25/development/tokenizer-options/ · https://blog.vectorchord.ai/vectorchord-10-developer-first-vector-search-on-postgres-100x-faster-indexing-than-pgvector
- Gemini 3.1 Flash-Lite pricing: https://blog.google/innovation-and-ai/technology/developers-tools/gemini-3-1-flash-lite/
- GPU hourly reference: https://www.morphllm.com/deepinfra-pricing

Content from these sources was paraphrased for compliance with licensing restrictions.
