# Architecture decision records

Short-form ADRs for the PoC. Each ADR has a status line and five sub-items: **.1 Context**, **.2 Decision**, **.3 Consequences**, **.4 Alternatives** and **.5 Revisit when**. Status values: Accepted, Superseded (with a link to the successor) and Retired ([CONV-01.3](../README.md#conv-01-3)). An ADR owns the rationale for a choice. What the chosen component does is owned by the linked ARCH, DEP, PORT, ADP, WF, ING or LLM item ([CONV-04.4](../README.md#conv-04-4)).

<a id="adr-01"></a>
### ADR-01 Hexagonal architecture with light command/query separation

Status: Accepted, 2026-11-10.

- <a id="adr-01-1"></a>**ADR-01.1 Context.** The PoC combines many volatile integrations: three source connectors, five extractors, a local LLM, an external embedding API, a workflow engine and three inbound channels (HTTP/UI, CLI, MCP). Several of them are expected to be swapped after the PoC: the embedding provider, blob storage, authentication and possibly the search engine. Reads (search, facets, overviews) and writes (ingest, assign, override) have very different shapes.
- <a id="adr-01-2"></a>**ADR-01.2 Decision.** Use ports and adapters, with a domain core that does no I/O. In the application layer, command handlers and query handlers are separate: commands mutate aggregates and queries read through a read model on the same database. There is no separate read store, no event sourcing and no replay. The layers are defined in [ARCH-02](overview.md#arch-02), the dependency rule in [ARCH-03](overview.md#arch-03) and the command/query split in [ARCH-04](overview.md#arch-04).
- <a id="adr-01-3"></a>**ADR-01.3 Consequences.** Each external technology sits behind a port, so swaps are adapter changes and are tested by port contract tests ([XC-07](overview.md#xc-07)). Queries can use hand-written SQL and views without going through aggregates. The cost is extra indirection (protocols, DTOs, mapping code) that a PoC of this size would otherwise skip. Because there is no event store, history comes only from the audit log ([ADR-13](#adr-13)).
- <a id="adr-01-4"></a>**ADR-01.4 Alternatives.** A layered or transaction-script design is quicker to start, but couples the business logic to SQLAlchemy, DBOS and HTTP, which are exactly the parts we expect to replace. Full CQRS with event sourcing was rejected by the user. It also buys nothing here, because the index can be rebuilt from its sources ([SCOPE-07](../README.md#scope-07)). Vertical slices without ports leave the adapter swaps above unprotected.
- <a id="adr-01-5"></a>**ADR-01.5 Revisit when** read load needs a separate store (for example a split-out search engine, see [ADR-03.5](#adr-03-5)). Even then the read model gains a sync path, but the commands do not change.

Refs: [ARCH-02](overview.md#arch-02), [ARCH-03](overview.md#arch-03), [ARCH-04](overview.md#arch-04), [SCOPE-08](../README.md#scope-08)

<a id="adr-02"></a>
### ADR-02 Python 3.12, FastAPI, Pydantic, SQLAlchemy Core

Status: Accepted, 2026-11-10.

- <a id="adr-02-1"></a>**ADR-02.1 Context.** The critical libraries are Python-first or Python-only: e-mail and PST parsing (extract-msg, pypff), OCRmyPDF, the official MCP SDK, DBOS Transact, and the Voyage and OpenAI clients. The team needs one language across workers, API and MCP server.
- <a id="adr-02-2"></a>**ADR-02.2 Decision.** Use Python 3.12 in a single uv-managed repository with one package, `docstore`. FastAPI and Pydantic v2 serve HTTP and the DTOs. Persistence uses SQLAlchemy 2.0 Core (not the ORM) with psycopg 3, and Alembic handles migrations. The module layout is in [MOD-01](overview.md#mod-01).
- <a id="adr-02-3"></a>**ADR-02.3 Consequences.** Choosing Core over the ORM keeps domain classes plain: repositories map rows explicitly, and the read model is plain SQL ([ADP-08](ports-adapters.md#adp-08)). CPU-heavy work (OCR, parsing) runs in worker processes and subprocesses, not threads, so the GIL is not a bottleneck. Python's weaker static guarantees are offset by Pydantic validation at the boundaries and typed `Protocol` ports.
- <a id="adr-02-4"></a>**ADR-02.4 Alternatives.** Go has better concurrency and a single binary, but lacks mature MSG, PST and OCR tooling and has no DBOS SDK at the same level. TypeScript has a strong MCP SDK but weak legal-format extraction. The SQLAlchemy ORM with imperative mapping is workable but hides the SQL that hybrid search and permission filters depend on. Django's ORM and app model conflict with the hexagonal layering.
- <a id="adr-02-5"></a>**ADR-02.5 Revisit when** a component becomes CPU-bound in Python in a way that worker scaling cannot absorb. In that case the component is replaced behind its port; the stack stays.

Refs: [MOD-01](overview.md#mod-01), [ADP-01](ports-adapters.md#adp-01), [ADP-08](ports-adapters.md#adp-08)

<a id="adr-03"></a>
### ADR-03 PostgreSQL with pgvector and pg_search as the single store

Status: Accepted, 2026-11-10.

- <a id="adr-03-1"></a>**ADR-03.1 Context.** Search must combine keyword, vector and metadata filters with a permission filter that is never bypassed ([F-13](../features.md#f-13), [F-16](../features.md#f-16)). The prior research showed that 100 GB is about 10M chunks, which a single Postgres server can carry, and that the hard part is metadata and ACLs, not vectors. A PoC benefits from one operational component.
- <a id="adr-03-2"></a>**ADR-03.2 Decision.** PostgreSQL 17, from the ParadeDB image, is the single system of record for metadata, assignments, ACLs, the audit log, BM25 (pg_search) and vectors (pgvector). Hybrid search is one SQL statement: the permission pre-filter, a BM25 leg and a vector leg, fused by RRF ([ADP-10](ports-adapters.md#adp-10)). Postgres `tsvector` full-text search is the fallback keyword leg if pg_search is unavailable or its licence is unacceptable. The same instance hosts the DBOS system database ([ADR-04](#adr-04), [DEP-02](overview.md#dep-02)).
- <a id="adr-03-3"></a>**ADR-03.3 Consequences.** ACL metadata has one source of truth, with no sync and no second copy to leak from. Index writes are transactional with metadata. A single instance is also a single point of failure and a shared resource for ingestion and search, which is accepted for the PoC ([SCOPE-06](../README.md#scope-06)). pg_search is AGPL-3.0: this is acceptable for internal use, and the `tsvector` fallback limits lock-in. The vector memory footprint is decided in [ADR-12](#adr-12).
- <a id="adr-03-4"></a>**ADR-03.4 Alternatives.** OpenSearch provides BM25, k-NN and document-level security in one engine, but becomes a second source of truth for ACLs next to Postgres. Qdrant or Weaviate with Postgres means two stores, an ACL sync and cross-store fusion. Elasticsearch's RRF retriever is an Enterprise-licensed feature. pgvectorscale (DiskANN) is a later option for the same store, not an alternative store.
- <a id="adr-03-5"></a>**ADR-03.5 Revisit when** the corpus grows beyond roughly 20–50M chunks, when concurrent agent load pushes hybrid p95 past [SC-07](../success-criteria.md#sc-07), or when ingestion writes visibly degrade search latency. The scale path then is Postgres for metadata and ACLs, plus a dedicated vector or keyword engine behind [PORT-10](ports-adapters.md#port-10) and [PORT-11](ports-adapters.md#port-11).

Refs: [ADP-08](ports-adapters.md#adp-08), [ADP-09](ports-adapters.md#adp-09), [ADP-10](ports-adapters.md#adp-10), [DEP-02](overview.md#dep-02)

<a id="adr-04"></a>
### ADR-04 DBOS Transact as the durable workflow engine

Status: Accepted, 2026-11-10.

- <a id="adr-04-1"></a>**ADR-04.1 Context.** A full ingestion of 100 GB runs for days, and enrichment on the local LLM dominates that time ([ADR-05.3](#adr-05-3)). The run must survive worker and database restarts, resume without redoing completed steps, never create duplicates, cap concurrency on the GPU and OCR, and respect the embedding API's rate limits ([F-04](../features.md#f-04), [SC-08](../success-criteria.md#sc-08), [SC-09](../success-criteria.md#sc-09)). The PoC should not add another stateful service to operate.
- <a id="adr-04-2"></a>**ADR-04.2 Decision.** Use DBOS Transact (Python, MIT licence). Workflows and steps are checkpointed in a DBOS system database on the existing Postgres instance. DBOS queues provide per-queue concurrency caps and rate limiters. Workflow IDs are deterministic, so starting the same work twice is a no-op. The engine is an inbound adapter whose steps call application commands ([ARCH-08](overview.md#arch-08), [ADP-05](ports-adapters.md#adp-05)). The application controls it through [PORT-18](ports-adapters.md#port-18). The workflows are defined in [WF-01](ingestion.md#wf-01) to [WF-05](ingestion.md#wf-05).
- <a id="adr-04-3"></a>**ADR-04.3 Consequences.**
  - No extra infrastructure is needed: the workers are the engine, and the only dependency is Postgres.
  - Workflow functions must be deterministic, and every step must be safe to re-execute ([ING-18](ingestion.md#ing-18)), because a step that crashed mid-way is run again.
  - In open-source DBOS, a crashed process's pending workflows are recovered by a process that restarts with the same executor ID. Each worker replica therefore gets a stable executor ID and the compose restart policy `always` ([DEP-05](overview.md#dep-05)). Automatic failover of one replica's workflows to another (DBOS Conductor) is out of scope.
  - One child workflow per item means millions of workflow rows at 100 GB. Discovery enqueues in bounded batches ([WF-01](ingestion.md#wf-01)), and completed workflow records are pruned under a retention setting ([XC-04](overview.md#xc-04)).
  - DBOS code is confined to [ADP-05](ports-adapters.md#adp-05) and [ADP-06](ports-adapters.md#adp-06), so the domain and application layers stay engine-agnostic.
- <a id="adr-04-4"></a>**ADR-04.4 Alternatives.**
  - Temporal has the strongest durability model and tooling, but needs a separate server cluster with its own persistence and a new programming model: too heavy for a local PoC.
  - Prefect 3 suits flow-level orchestration, but needs a server and has awkward per-item granularity at millions of tasks.
  - Hatchet is Postgres-backed but runs as a separate engine service.
  - Procrastinate (a Postgres job queue) gives durable jobs, but no step checkpoints or workflow state, so we would write the state machine ourselves.
  - Celery with Redis provides neither durable step state nor first-class rate limiting.
  - A hand-rolled state machine over the source-item table is possible, but re-implements retries, recovery and throttling.
- <a id="adr-04-5"></a>**ADR-04.5 Revisit when** multi-host workers need automatic cross-host recovery, when the system database's write volume measurably competes with search on the shared instance, or when an operations team requires an orchestration UI. Candidates are DBOS Conductor, a separate Postgres for the system database, or Temporal behind the same [PORT-18](ports-adapters.md#port-18).

Refs: [WF-01](ingestion.md#wf-01), [WF-04](ingestion.md#wf-04), [WF-05](ingestion.md#wf-05), [ADP-05](ports-adapters.md#adp-05), [ADP-06](ports-adapters.md#adp-06), [ARCH-08](overview.md#arch-08)

<a id="adr-05"></a>
### ADR-05 Local LLM via Ollama for enrichment

Status: Accepted, 2026-11-10.

- <a id="adr-05-1"></a>**ADR-05.1 Context.** The user decided that ingestion enriches every document with an LLM, producing tags, document type, case-type suggestion, parties, dates and a summary ([F-09](../features.md#f-09)). This reverses the research's zero-LLM recommendation. Full document text is far more sensitive than chunk embeddings, so the LLM must run on the local machine. The PoC host has at most one consumer GPU.
- <a id="adr-05-2"></a>**ADR-05.2 Decision.** Serve the model with Ollama through its OpenAI-compatible endpoint. The default model is Qwen3-14B-Instruct, quantised Q4_K_M, on a 24 GB GPU. Qwen3-8B is the fallback for CPU-only or smaller GPUs. Calls use temperature 0 and JSON-schema structured output. vLLM is a drop-in alternative behind the same [PORT-13](ports-adapters.md#port-13), because both expose the OpenAI API. The LLM is used only at ingest ([SCOPE-05](../README.md#scope-05)). The runtime contract is in [LLM-01](ingestion.md#llm-01).
- <a id="adr-05-3"></a>**ADR-05.3 Consequences.**
  - No document text reaches a third party through enrichment.
  - At a few seconds per document on one GPU, enrichment is expected to be the slowest stage and therefore the bottleneck of a 100 GB run. The measured rate feeds the projection in [SC-09](../success-criteria.md#sc-09) and [DEP-08](overview.md#dep-08).
  - Work avoidance matters: exact duplicates are enriched only once, and inputs are bounded excerpts ([LLM-02](ingestion.md#llm-02)).
  - Switching to vLLM with continuous batching is the main throughput lever.
  - Qwen3's multilingual ability fits mixed-language firm content. Quantisation and model choice must be pinned and stamped in provenance ([LLM-04](ingestion.md#llm-04)), because a model change invalidates enrichment ([XC-06](overview.md#xc-06)).
- <a id="adr-05-4"></a>**ADR-05.4 Alternatives.**
  - Zero-LLM ingestion (the research recommendation) cannot produce semantic tags, document types or summaries.
  - A hosted LLM API sends whole documents off-premises, which is ruled out by [SCOPE-04](../README.md#scope-04).
  - Llama 3.1 8B and Mistral Small showed weaker multilingual extraction and weaker schema adherence at comparable size.
  - Running llama.cpp server directly gives less model management.
  - Making vLLM the default needs more VRAM headroom and set-up than a PoC laptop or workstation reliably has.
- <a id="adr-05-5"></a>**ADR-05.5 Revisit when** the measured enrichment rate makes the projected 100 GB duration unacceptable (move to vLLM, a larger GPU or selective enrichment), or when gold-set quality misses [SC-04](../success-criteria.md#sc-04) (try a larger model or prompt changes under [LLM-04](ingestion.md#llm-04)).

Refs: [ADP-13](ports-adapters.md#adp-13), [DEP-07](overview.md#dep-07), [LLM-01](ingestion.md#llm-01), [SCOPE-05](../README.md#scope-05)

<a id="adr-06"></a>
### ADR-06 Embeddings via external API behind a port

Status: Accepted, 2026-11-10.

- <a id="adr-06-1"></a>**ADR-06.1 Context.** The user decided that embeddings are computed by an external API. On a single-GPU PoC host, the GPU is already saturated by enrichment ([ADR-05.3](#adr-05-3)), so a local embedding model would compete with it. Hosted embedding models give strong multilingual retrieval quality without any operational work. The research's preferred option was local embeddings, chosen for confidentiality.
- <a id="adr-06-2"></a>**ADR-06.2 Decision.**
  - Embeddings are computed through [PORT-12](ports-adapters.md#port-12).
  - The default is Voyage `voyage-3.5` (1024 dimensions, multilingual), called with separate document and query input types ([ADP-11](ports-adapters.md#adp-11)).
  - The alternates are `voyage-law-2` and OpenAI `text-embedding-3-large` with `dimensions=1024` ([ADP-12](ports-adapters.md#adp-12)), so the vector column stays 1024-d.
  - Every chunk is stamped with the model ID and dimension, and queries always use the model of the indexed vectors.
  - We accept that chunk text and query text leave the machine. The PoC is therefore restricted to sample or consented data ([SCOPE-04](../README.md#scope-04)).
- <a id="adr-06-3"></a>**ADR-06.3 Consequences.**
  - Egress must be limited to the provider host ([XC-02](overview.md#xc-02)).
  - Rate limits and 429 responses are a normal operating condition and are handled by the embed queue limiter ([WF-04](ingestion.md#wf-04)).
  - A full 100 GB pass has a token cost per the projection of [DEP-08.6](overview.md#dep-08-6).
  - The query-time vector leg depends on the API being reachable. If it is not, search must degrade to the keyword leg rather than fail ([QRY-01](application.md#qry-01)).
  - Changing the model means re-embedding every chunk ([ADR-12](#adr-12), [ING-17](ingestion.md#ing-17)).
- <a id="adr-06-4"></a>**ADR-06.4 Alternatives.**
  - A local bge-m3 under Text Embeddings Inference keeps all text on premises, but competes with the LLM for the GPU. It is the planned production swap behind [PORT-12](ports-adapters.md#port-12).
  - OpenAI `text-embedding-3-small` is cheaper but has lower retrieval quality.
  - Cohere embed-multilingual is a comparable alternate, but no adapter is planned.
  - Using the LLM's own hidden states as embeddings gives poor retrieval quality.
- <a id="adr-06-5"></a>**ADR-06.5 Revisit when** any non-consented client data is to be processed, when the firm's policy or DPIA rules out off-premises processing of document text, or when the PoC moves to production. The default answer then is a local embedding adapter plus a full re-embed.

Refs: [PORT-12](ports-adapters.md#port-12), [ADP-11](ports-adapters.md#adp-11), [ADP-12](ports-adapters.md#adp-12), [XC-02](overview.md#xc-02), [SCOPE-04](../README.md#scope-04)

<a id="adr-07"></a>
### ADR-07 Extraction with Tika, OCRmyPDF and dedicated e-mail parsers

Status: Accepted, 2026-11-10.

- <a id="adr-07-1"></a>**ADR-07.1 Context.** The corpus mixes legacy and modern Office formats, born-digital and scanned PDFs, and e-mail as PST, MSG and EML ([SCOPE-03](../README.md#scope-03)). Format coverage of at least 98% with no silent drops ([SC-01](../success-criteria.md#sc-01)) and searchable scans ([SC-02](../success-criteria.md#sc-02)) are acceptance criteria. Faithful e-mail headers are needed for threading and dedup ([F-07](../features.md#f-07)).
- <a id="adr-07-2"></a>**ADR-07.2 Decision.**
  - The Apache Tika server handles detection and text and metadata extraction for Office (including legacy DOC, XLS and PPT), PDF text layers, RTF and generic formats, with embedded documents ([ADP-14](ports-adapters.md#adp-14)).
  - OCRmyPDF with Tesseract, in skip-text mode, OCRs only pages without a text layer ([ADP-16](ports-adapters.md#adp-16)).
  - E-mail uses the Python `email` package for EML and extract-msg for MSG ([ADP-15](ports-adapters.md#adp-15)), and pypff (libpff) for PST ([ADP-18](ports-adapters.md#adp-18)).
- <a id="adr-07-3"></a>**ADR-07.3 Consequences.**
  - Tika runs as a separate JVM service ([DEP-06](overview.md#dep-06)), so a crash or out-of-memory error in a parser is isolated from the workers.
  - OCR is CPU-heavy and runs in the worker image under a concurrency cap ([WF-04](ingestion.md#wf-04)).
  - pypff gives message-level access with stable keys, so PST processing is resumable per message rather than per archive.
  - Licences: extract-msg is GPL-3.0 and libpff is LGPL-3.0. Both are acceptable for internal PoC use and must be reviewed before any distribution.
  - Each extractor reports its version, which is stamped on the extracted text ([XC-06](overview.md#xc-06)).
- <a id="adr-07-4"></a>**ADR-07.4 Alternatives.**
  - Unstructured is heavy and has gaps in legacy formats.
  - Docling has strong PDF layout handling, but is slower and does not cover legacy Office.
  - LibreOffice headless conversion to modern formats adds a fragile conversion hop.
  - Tika's own MSG and PST parsers have lower header fidelity, and their whole-archive PST handling is not resumable.
  - readpst (libpst) export to EML is kept as a manual fallback for PSTs that pypff cannot open.
- <a id="adr-07-5"></a>**ADR-07.5 Revisit when** layout-sensitive extraction (tables in contracts, pleadings structure) becomes a requirement for chunking quality, at which point Docling becomes a candidate adapter for PDF, or when the quarantine rate of a format class exceeds the [SC-01](../success-criteria.md#sc-01) budget.

Refs: [ADP-14](ports-adapters.md#adp-14), [ADP-15](ports-adapters.md#adp-15), [ADP-16](ports-adapters.md#adp-16), [ADP-18](ports-adapters.md#adp-18), [DEP-06](overview.md#dep-06)

<a id="adr-08"></a>
### ADR-08 Rules-first assignment with LLM as suggester and human override

Status: Accepted, 2026-11-10.

- <a id="adr-08-1"></a>**ADR-08.1 Context.** A document's client and matter assignment decides who may see it. A wrong auto-assignment can leak a document across an ethical wall, so assignment errors are both a confidentiality problem and a quality problem. Strong deterministic signals exist: folder paths, SharePoint sites, matter codes in file names and subjects, client e-mail domains and mailbox owners. The practice-management system (PMS) is the authority for clients and matters. LLM output is useful but can hallucinate.
- <a id="adr-08-2"></a>**ADR-08.2 Decision.** Deterministic rules from a versioned rule set ([DM-15](domain.md#dm-15)), together with inheritance from the family or thread, are the primary assignment method. The LLM only proposes candidates and adjusts confidence: an assignment backed only by the LLM never auto-accepts. Confidence bands decide between auto-accept and review ([TAX-04](domain.md#tax-04)). Human decisions take precedence and survive reprocessing; the precedence invariant is in [DM-10](domain.md#dm-10). Reconciliation of the signals is [CMD-12](application.md#cmd-12).
- <a id="adr-08-3"></a>**ADR-08.3 Consequences.** Auto-accepted assignments are explainable, because each one cites a rule version. Precision is protected at the cost of review volume. The review queue ([F-10](../features.md#f-10)) is therefore a core PoC feature, not an extra. Rule quality becomes the main tuning lever, so rule sets are versioned and re-application is version-driven ([XC-06](overview.md#xc-06)).
- <a id="adr-08-4"></a>**ADR-08.4 Alternatives.** LLM-first classification gives higher recall but unexplainable errors that can breach walls. A supervised classifier trained on PMS-labelled documents is promising, but there are not enough labels in the PoC. It is a later option that can sit behind the same reconciliation. Manual-only assignment does not scale to 2M documents.
- <a id="adr-08-5"></a>**ADR-08.5 Revisit when** review volume on the sample makes review at 100 GB impractical despite [SC-03](../success-criteria.md#sc-03) precision. Options then are to tighten the rules, add a trained classifier as another signal, or allow LLM-only auto-accept for low-risk roles such as case type, but not for client or matter.

Refs: [DM-10](domain.md#dm-10), [DM-15](domain.md#dm-15), [ING-09](ingestion.md#ing-09), [ING-11](ingestion.md#ing-11), [CMD-12](application.md#cmd-12)

<a id="adr-09"></a>
### ADR-09 Permission filtering before retrieval in SQL

Status: Accepted, 2026-11-10.

- <a id="adr-09-1"></a>**ADR-09.1 Context.** Ethical walls must hide both a matter's documents and the fact that the matter exists ([GL-13](../README.md#gl-13)). Leaks happen through result lists, but also through facet counts, total hits, "similar documents", thread views and agent tools ([SC-12](../success-criteria.md#sc-12)). Post-filtering a top-k list also loses recall when a principal can see only a small slice of the corpus.
- <a id="adr-09-2"></a>**ADR-09.2 Decision.** Every read-model query takes the principal's permission predicate and applies it inside the SQL, before ranking, counting and fusion ([XC-01](overview.md#xc-01), [ADP-10](ports-adapters.md#adp-10)). Access keys are denormalised onto chunks so that both search legs can filter without joins ([DM-05](domain.md#dm-05)). Postgres row-level security is a backstop in case a query forgets the predicate. It is not the primary mechanism.
- <a id="adr-09-3"></a>**ADR-09.3 Consequences.** Walled content is indistinguishable from nonexistent content on every path, including MCP ([MCP-02](mcp.md#mcp-02)). Filtered vector search needs pgvector's iterative index scans to keep recall with selective filters; recall is verified by [SC-05](../success-criteria.md#sc-05) for restricted principals. Grant and wall changes need no key update, because keys are matter or source IDs and visibility is resolved per query ([CMD-22.3](application.md#cmd-22-3)); only assignment changes rewrite keys ([DM-03.14](domain.md#dm-03-14)). RLS adds planner overhead and must be checked in the latency tests ([SC-06](../success-criteria.md#sc-06)).
- <a id="adr-09-4"></a>**ADR-09.4 Alternatives.** Post-filtering in the application leaks counts and degrades recall. One index per matter or access group breaks cross-matter search and multiplies index memory. RLS alone is hard to reason about, makes plans opaque and is easy to defeat with a privileged connection. Search-engine document-level security is only relevant if a second engine is introduced ([ADR-03.5](#adr-03-5)).
- <a id="adr-09-5"></a>**ADR-09.5 Revisit when** the number of distinct access keys per principal grows so large that the predicate degrades plans, in which case precomputed principal-to-key sets are an option, or when a second search engine is introduced.

Refs: [XC-01](overview.md#xc-01), [ADP-10](ports-adapters.md#adp-10), [MCP-02](mcp.md#mcp-02)

<a id="adr-10"></a>
### ADR-10 MCP via official Python SDK with PoC token auth

Status: Accepted, 2026-11-10.

- <a id="adr-10-1"></a>**ADR-10.1 Context.** Agents must reach the store over MCP with the same permissions as the person they act for ([F-18](../features.md#f-18)). Generic database and search-engine MCP servers do not enforce per-user ACLs. The MCP authorisation specification (OAuth 2.1 resource server) is the production target, but it needs an identity provider integration that the PoC does not have.
- <a id="adr-10-2"></a>**ADR-10.2 Decision.** Build a thin server on the official `mcp` Python SDK (FastMCP), using Streamable HTTP in stateless mode, as a separate process that calls the application handlers in-process ([MCP-01](mcp.md#mcp-01), [ADP-04](ports-adapters.md#adp-04)). Authentication uses static bearer tokens configured per agent and principal, mapped to principals ([ADP-07](ports-adapters.md#adp-07)). The tool set is read-only except for flag-for-review ([MCP-09](mcp.md#mcp-09)).
- <a id="adr-10-3"></a>**ADR-10.3 Consequences.** The tools reuse the queries and the permission predicate unchanged, so MCP cannot be more permissive than the UI. Stateless transport allows several replicas without session affinity. Static tokens are suitable only for a local PoC: revocation needs a file edit and restart ([ADP-07.3](ports-adapters.md#adp-07-3), [SCOPE-06](../README.md#scope-06)). MCP specification revisions need ongoing maintenance.
- <a id="adr-10-4"></a>**ADR-10.4 Alternatives.** stdio transport is single-user and local to one client, and cannot be shared. The archived reference Postgres MCP server and the Qdrant or OpenSearch MCP servers expose raw data without firm ACLs. Implementing OAuth 2.1 with OIDC now is too costly for the PoC. Exposing only the REST API to agents loses MCP tool discovery.
- <a id="adr-10-5"></a>**ADR-10.5 Revisit when** the PoC is used beyond a single machine or with real users. The token authenticator is then replaced by an OAuth 2.1 resource-server adapter behind [PORT-03](ports-adapters.md#port-03).

Refs: [MCP-01](mcp.md#mcp-01), [MCP-02](mcp.md#mcp-02), [ADP-04](ports-adapters.md#adp-04), [ADP-07](ports-adapters.md#adp-07), [SCOPE-06](../README.md#scope-06)

<a id="adr-11"></a>
### ADR-11 Content-addressed local blob storage

Status: Accepted, 2026-11-10.

- <a id="adr-11-1"></a>**ADR-11.1 Context.** Reprocessing (new parser, prompt or model) must not depend on sources staying reachable or unchanged, and PST archives are expensive to reopen. Users need to download originals. Duplicates are frequent in legal corpora, especially e-mail attachments.
- <a id="adr-11-2"></a>**ADR-11.2 Decision.** Originals and extracted text are stored on a local filesystem volume, keyed by sha256 under a fan-out directory layout ([ADP-20](ports-adapters.md#adp-20)), behind [PORT-17](ports-adapters.md#port-17). MinIO or S3 is a later swap behind the same port.
- <a id="adr-11-3"></a>**ADR-11.3 Consequences.** Byte-identical content is stored once, and writes are idempotent because the key is the content. Reprocessing reads local copies. The store holds a copy of confidential originals, so it falls under the PoC data policy ([SCOPE-04](../README.md#scope-04)). Removing a document requires reference counting across duplicates, which is deferred along with hard-delete ([SCOPE-06](../README.md#scope-06)). The volume size is part of [DEP-08](overview.md#dep-08).
- <a id="adr-11-4"></a>**ADR-11.4 Alternatives.** Keeping references only and re-fetching from the source makes reprocessing slow and non-reproducible when sources change. MinIO from day one adds a service with no PoC benefit. Postgres `bytea` or large objects would bloat the database and its backups.
- <a id="adr-11-5"></a>**ADR-11.5 Revisit when** storage must be shared across hosts, needs object lock or retention, or outgrows a single volume.

Refs: [PORT-17](ports-adapters.md#port-17), [ADP-20](ports-adapters.md#adp-20), [SCOPE-07](../README.md#scope-07)

<a id="adr-12"></a>
### ADR-12 Chunking and vector index configuration

Status: Accepted, 2026-11-10.

- <a id="adr-12-1"></a>**ADR-12.1 Context.** About 10M chunks at 100 GB must fit in one Postgres host's memory with acceptable recall and latency ([SC-07](../success-criteria.md#sc-07)). Legal text has long clauses, so chunks that are too small lose context and chunks that are too large blur similarity. Embedding models will change over the life of the store.
- <a id="adr-12-2"></a>**ADR-12.2 Decision.**
  - Chunks target about 512 tokens with about 64 tokens of overlap, preferring paragraph and heading boundaries ([ING-12](ingestion.md#ing-12)).
  - Vectors are stored as pgvector `halfvec(1024)` with an HNSW index (cosine). Build parameters start at m=16 and ef_construction=64, and `ef_search` and iterative scan are tuned against [SC-05](../success-criteria.md#sc-05) and [SC-06](../success-criteria.md#sc-06) ([ADP-09](ports-adapters.md#adp-09)).
  - The chunker version and embedding model are stamped on every chunk ([DM-05](domain.md#dm-05), [XC-06](overview.md#xc-06)).
- <a id="adr-12-3"></a>**ADR-12.3 Consequences.** Half precision halves vector storage and index memory compared with `vector(1024)`, with negligible recall loss for retrieval. Projected sizes are in [DEP-08](overview.md#dep-08). A new chunker version means re-chunking and re-embedding. A new embedding model means re-embedding all chunks, during which queries must use the model that matches the indexed vectors ([ING-17](ingestion.md#ing-17)). Building an HNSW index over 10M vectors takes hours, so bulk loads build the index after loading.
- <a id="adr-12-4"></a>**ADR-12.4 Alternatives.** 256-token chunks double the vector count and fragment clauses. 1024-token chunks dilute similarity and inflate prompts for downstream agents. Full-precision `vector(1024)` doubles the memory. IVFFlat needs training and recalls worse under filters. Semantic or layout-aware chunking is later work ([ADR-07.5](#adr-07-5)). Binary or int8 quantisation and pgvectorscale's DiskANN are scale levers kept in reserve.
- <a id="adr-12-5"></a>**ADR-12.5 Revisit when** the synthetic load in [SC-07](../success-criteria.md#sc-07) misses its p95 target or the index no longer fits in RAM, in which case quantisation or DiskANN come next, or when relevance evaluation shows chunk size as the limiting factor.

Refs: [DM-05](domain.md#dm-05), [ING-12](ingestion.md#ing-12), [ADP-09](ports-adapters.md#adp-09), [DEP-08](overview.md#dep-08)

<a id="adr-13"></a>
### ADR-13 In-process domain events, same-database audit

Status: Accepted, 2026-11-10.

- <a id="adr-13-1"></a>**ADR-13.1 Context.** Some side effects follow state changes: an audit entry, opening a review task, and refreshing progress counters. Full CQRS and event sourcing are out of scope ([SCOPE-08](../README.md#scope-08)), and there is a single process type per container with a single database.
- <a id="adr-13-2"></a>**ADR-13.2 Decision.** Aggregates record domain events, and the unit of work dispatches them synchronously after commit, in the same process, to in-process handlers ([ARCH-05](overview.md#arch-05), [ADP-21](ports-adapters.md#adp-21)). Audit goes to an append-only table in the same database: entries for commands of human and agent principals are written inside the command transaction by the unit of work, and only worker events go through the post-commit audit subscriber ([XC-08.2](overview.md#xc-08-2)). There is no broker, no outbox and no replay.
- <a id="adr-13-3"></a>**ADR-13.3 Consequences.** The mechanism is simple and debuggable, with no infrastructure. If the process crashes between commit and dispatch, side effects can be lost. This is accepted because ingestion steps are re-executed by the workflow engine and handlers are idempotent ([ING-18](ingestion.md#ing-18)). Audit rows for human and agent commands cannot be lost this way, because they commit with the change. Query and MCP access auditing is a separate path that does not depend on events ([XC-08](overview.md#xc-08)).
- <a id="adr-13-4"></a>**ADR-13.4 Alternatives.** A transactional outbox with a relay guarantees delivery but adds a poller and its own failure modes. A message broker (Redis Streams, RabbitMQ) is a new service with no consumer outside the process. Writing every audit row, worker events included, inside the command transaction would add a write to each pipeline step for little gain.
- <a id="adr-13-5"></a>**ADR-13.5 Revisit when** audit completeness for worker events becomes a hard requirement (write in-transaction or add an outbox), or when an external consumer of events appears.

Refs: [ARCH-05](overview.md#arch-05), [PORT-19](ports-adapters.md#port-19), [ADP-21](ports-adapters.md#adp-21), [XC-08](overview.md#xc-08)

<a id="adr-14"></a>
### ADR-14 Minimal server-rendered UI and CLI

Status: Accepted, 2026-11-10.

- <a id="adr-14-1"></a>**ADR-14.1 Context.** The PoC needs human screens for search, document view, the review queue and run progress ([F-21](../features.md#f-21)), plus scripted administration and evaluation. UI polish is not a success criterion, but the review queue and progress view are ([SC-03](../success-criteria.md#sc-03), [SC-10](../success-criteria.md#sc-10)).
- <a id="adr-14-2"></a>**ADR-14.2 Decision.** Server-rendered Jinja2 templates with HTMX for partial updates are served by the API process ([ADP-02](ports-adapters.md#adp-02)). A Typer CLI is used for administration, runs, imports and evaluation ([ADP-03](ports-adapters.md#adp-03)). Both are inbound adapters over the same command and query handlers.
- <a id="adr-14-3"></a>**ADR-14.3 Consequences.** There is no JavaScript build chain and no separate front-end deployment, and the UI reuses Python DTOs directly. Progress views poll via HTMX. Rich interactions (document viewer annotations, bulk review) stay limited.
- <a id="adr-14-4"></a>**ADR-14.4 Alternatives.** A React or Vue SPA is a separate build and deployment with no PoC benefit. Streamlit or Gradio is quick but sits outside the hexagon, has weak auth and pulls logic into scripts. An API-only approach has no review UX for the assignment success criteria.
- <a id="adr-14-5"></a>**ADR-14.5 Revisit when** the store moves towards production users or the review workload needs bulk, keyboard-driven workflows.

Refs: [ADP-02](ports-adapters.md#adp-02), [ADP-03](ports-adapters.md#adp-03), [F-21](../features.md#f-21)
