# PoC success criteria

This file owns the success criteria (SC-) of the proof of concept: what must be true for the PoC to count as successful, how each criterion is measured and the target values. Features that serve a criterion are in [features.md](features.md). Scope and non-goals are in [README.md](README.md#scope-01). ID and link conventions follow [CONV-01](README.md#conv-01) to [CONV-05](README.md#conv-05).

Each criterion is a set of acceptance checks (SC-NN.M). Each check gives a **Target** and a **Method**. A check tagged **(hard)** is a gate; all other checks are targets. The verdict rule is [SC-15.7](#sc-15-7). All measurements use the datasets and conditions in [SC-15](#sc-15).

| ID | Criterion | Hard checks |
|---|---|---|
| [SC-01](#sc-01) | Format coverage | [SC-01.2](#sc-01-2) |
| [SC-02](#sc-02) | OCR of scanned documents | none |
| [SC-03](#sc-03) | Assignment accuracy and review routing | [SC-03.4](#sc-03-4) |
| [SC-04](#sc-04) | LLM enrichment quality and provenance | [SC-04.3](#sc-04-3), [SC-04.6](#sc-04-6), [SC-04.7](#sc-04-7) |
| [SC-05](#sc-05) | Search relevance | none |
| [SC-06](#sc-06) | Search latency | none |
| [SC-07](#sc-07) | Scale readiness for 100 GB | [SC-07.4](#sc-07-4) |
| [SC-08](#sc-08) | Durable, resumable, idempotent ingestion | [SC-08.1](#sc-08-1), [SC-08.2](#sc-08-2), [SC-08.3](#sc-08-3), [SC-08.5](#sc-08-5) |
| [SC-09](#sc-09) | Throughput, back-pressure and GPU sharing | [SC-09.1](#sc-09-1), [SC-09.3](#sc-09-3) |
| [SC-10](#sc-10) | Progress observability | none |
| [SC-11](#sc-11) | Agent access via MCP | [SC-11.6](#sc-11-6) |
| [SC-12](#sc-12) | Permission correctness | [SC-12.2](#sc-12-2), [SC-12.3](#sc-12-3), [SC-12.8](#sc-12-8) |
| [SC-13](#sc-13) | Dedup and threading quality | [SC-13.1](#sc-13-1) |
| [SC-14](#sc-14) | Architecture conformance and reproducibility | [SC-14.1](#sc-14-1), [SC-14.6](#sc-14-6) |
| [SC-15](#sc-15) | Evaluation baseline and verdict | [SC-15.6](#sc-15-6) |
| [SC-16](#sc-16) | Local inference and two-stage ingestion | [SC-16.1](#sc-16-1), [SC-16.2](#sc-16-2), [SC-16.4](#sc-16-4) |

---

<a id="sc-01"></a>
### SC-01 Format coverage

At least 98% of sample items in each in-scope format class are indexed. The rest are quarantined or excluded with a stated reason, and nothing is dropped silently.

- <a id="sc-01-1"></a>**SC-01.1** Indexed share per format class. **Target:** ≥ 98% of manifest items per class reach the indexed state. Reporting classes are mapped from source kind ([DM-01.1](architecture/domain.md#dm-01-1)), `format_class` ([TAX-06](architecture/domain.md#tax-06)) and the scanned flag (`has_text_layer` false, [DM-03.3](architecture/domain.md#dm-03-3)): PST message (kind `pst`), EML and MSG (`email` from other sources, split by file extension), OOXML (`word`, `excel`, `powerpoint`), legacy Office (the `_legacy` classes), born-digital PDF (`pdf` with a text layer), scanned PDF/image (`pdf` without a text layer, or `image`), SharePoint item (kind `sharepoint`), local-drive file (kind `filesystem`). An item counts in its format class and in its source class. Items in the deliberate corrupt set of [SC-15.1](#sc-15-1) are excluded from the denominator. **Method:** the harness ([F-20](features.md#f-20)) joins the corpus manifest to document and source-item status by sha256 and source key, grouped by class.
- <a id="sc-01-2"></a>**SC-01.2** **(hard)** No silent drops. **Target:** after a run completes, 100% of manifest items and discovered children are in exactly one terminal state: indexed, duplicate of a canonical document, quarantined with an error class from [XC-05](architecture/overview.md#xc-05), or excluded by a named rule (stage status `excluded`, [DM-02.6](architecture/domain.md#dm-02-6)). Items with no terminal state: 0. **Method:** SQL reconciliation of manifest against [DM-02](architecture/domain.md#dm-02) states.
- <a id="sc-01-3"></a>**SC-01.3** Bad input does not stop the run. **Target:** 100% of the corrupt, encrypted and zero-byte set is quarantined with the correct error class, and the run still reaches completed. **Method:** compare expected class per file in the manifest with the failure list from [QRY-11](architecture/application.md#qry-11).
- <a id="sc-01-4"></a>**SC-01.4** Attachments and embedded files. **Target:** ≥ 98% of manifest attachments, including attachments of attached e-mails nested two levels deep, become child documents in the correct family ([GL-04](README.md#gl-04)). **Method:** attachment manifest (parent key, filename, sha256) matched against family links.
- <a id="sc-01-5"></a>**SC-01.5** Native metadata. **Target:** ≥ 95% of e-mails have the e-mail header field group of [DM-03](architecture/domain.md#dm-03) populated (sent date, sender, recipients, subject, Message-ID). For Office and PDF files, ≥ 95% of author and created/modified values present in the source are captured. **Method:** 100-document stratified spot check against values read with a reference tool (Outlook/`readpst`, file properties).
- <a id="sc-01-6"></a>**SC-01.6** Incremental discovery. **Target:** after a change set of 20 additions, 20 edits and 20 deletions across the local drive and SharePoint sources, a new run reflects 100% of the changes and reprocesses no unchanged item. SharePoint items keep site, library, path and column metadata. **Method:** change script plus diff of source-item fingerprints and stage counters ([ING-01](architecture/ingestion.md#ing-01), [ADP-19](architecture/ports-adapters.md#adp-19)).

Refs: [F-01](features.md#f-01), [F-02](features.md#f-02), [SCOPE-03](README.md#scope-03), [SC-15.1](#sc-15-1)

<a id="sc-02"></a>
### SC-02 OCR of scanned documents

Scanned PDFs and images become searchable. OCR runs only where no usable text layer exists.

- <a id="sc-02-1"></a>**SC-02.1** Searchability. **Target:** 100% of non-quarantined scanned items have non-empty text. For 50 distinctive-phrase probes (one per scanned document), the source document appears in the top 10 of keyword search in ≥ 95% of probes. **Method:** harness phrase-probe script against [QRY-01](architecture/application.md#qry-01) in keyword mode.
- <a id="sc-02-2"></a>**SC-02.2** Character accuracy. **Target:** 1 − CER ≥ 95% on the clean-scan subset (≥ 30 pages, ≥ 300 dpi, with ground-truth transcripts). Degraded scans (fax, skew, stamps) are reported separately without a target. **Method:** Levenshtein character error rate of OCR text against the transcripts, reported per configured Tesseract language.
- <a id="sc-02-3"></a>**SC-02.3** OCR only where needed. **Target:** ≤ 1% of born-digital PDFs are sent to OCR, and ≥ 98% of scanned items are detected as needing OCR. **Method:** OCR step invocations from step logs ([XC-03](architecture/overview.md#xc-03)) compared with the manifest's scanned flag.
- <a id="sc-02-4"></a>**SC-02.4** Mixed PDFs. **Target:** in ≥ 10 PDFs that mix text and image pages, 100% of image-only pages gain text and text-layer pages keep their original text unchanged. **Method:** per-page text diff before and after the OCR step ([ING-06](architecture/ingestion.md#ing-06)).

Refs: [F-03](features.md#f-03), [ADR-07](architecture/decisions.md#adr-07)

<a id="sc-03"></a>
### SC-03 Assignment accuracy and review routing

Documents are assigned to the right client, matter, handler and case type. Automatic acceptance is precise, and every uncertain case reaches a human.

- <a id="sc-03-1"></a>**SC-03.1** Client and matter accuracy. **Target:** ≥ 90% each on the gold set ([SC-15.2](#sc-15-2)), counting the highest-ranked assignment after stage 2 completes and before any human review. **Method:** harness compares [DM-10](architecture/domain.md#dm-10) assignments to gold labels. Documents with no proposal count as wrong.
- <a id="sc-03-2"></a>**SC-03.2** Auto-accept precision. **Target:** ≥ 97% precision for client and matter assignments that were auto-accepted under the confidence bands of [TAX-04](architecture/domain.md#tax-04). **Method:** precision over the auto-accepted subset of the gold set.
- <a id="sc-03-3"></a>**SC-03.3** Review load. **Target:** ≥ 60% of gold documents are auto-accepted for both client and matter, so that ≤ 40% need review. **Method:** share of gold documents by assignment status.
- <a id="sc-03-4"></a>**SC-03.4** **(hard)** Routing completeness. **Target:** 100% of documents that have no auto-accepted client and matter, or conflicting rule and LLM signals, have an open review task ([DM-13](architecture/domain.md#dm-13)), including documents with no matter candidate at all ([DM-10.6](architecture/domain.md#dm-10-6) rule 6). The check runs twice, once after stage 1 and once after stage 2 ([ING-21](architecture/ingestion.md#ing-21)), and both times unassigned documents without a task: 0. **Method:** SQL anti-join over the full sample corpus, run at both points.
- <a id="sc-03-5"></a>**SC-03.5** Handler and case type accuracy. **Target:** ≥ 85% each on the gold set. **Method:** same as [SC-03.1](#sc-03-1).
- <a id="sc-03-6"></a>**SC-03.6** Family and thread consistency. **Target:** 100% of attachments and thread members without a human override carry the same client and matter as their family parent or thread, as inheritance rules require. A later matter change reaches them through the durable reprocess run requested by the [EVT-04](architecture/domain.md#evt-04) handler ([WF-03.6](architecture/ingestion.md#wf-03-6)); the check runs after those runs complete. **Method:** SQL consistency check.
- <a id="sc-03-7"></a>**SC-03.7** Override takes effect. **Target:** after a reviewer resolves a task in the UI, the very next search filtered by the new matter returns the document, and its provenance shows method human. Median reviewer time per task is ≤ 30 s over 50 tasks. **Method:** scripted UI session timing plus a [QRY-01](architecture/application.md#qry-01) check after each resolution.
- <a id="sc-03-8"></a>**SC-03.8** Stage-1-only assignment. **Target:** the [SC-03.2](#sc-03-2) precision holds for client and matter assignments auto-accepted in stage 1. The share of gold documents with an accepted client and matter after stage 1 is reported, together with the stage-2 delta on [SC-03.1](#sc-03-1) and [SC-03.5](#sc-03-5). **Method:** harness snapshot after stage 1 (a run with `enrich_mode=none`) and after stage 2 ([ING-21](architecture/ingestion.md#ing-21)).

Refs: [F-08](features.md#f-08), [F-10](features.md#f-10), [F-12](features.md#f-12), [F-20](features.md#f-20), [ADR-08](architecture/decisions.md#adr-08)

<a id="sc-04"></a>
### SC-04 LLM enrichment quality and provenance

The local LLM produces useful, vocabulary-conformant enrichment. Every value is traceable, and human corrections are never overwritten.

- <a id="sc-04-1"></a>**SC-04.1** Document type accuracy. **Target:** top-1 accuracy ≥ 85% against gold labels from [TAX-02](architecture/domain.md#tax-02). **Method:** harness over [SC-15.2](#sc-15-2).
- <a id="sc-04-2"></a>**SC-04.2** Tag usefulness. **Target:** ≥ 80% of assigned tags are rated useful, and ≥ 90% of documents get at least one useful tag. **Method:** two lawyers rate each tag useful or not useful on a 100-document stratified sample. Disagreements count as not useful.
- <a id="sc-04-3"></a>**SC-04.3** **(hard)** Vocabulary conformance. **Target:** 100% of stored document types and case types are active taxonomy terms. Tags outside the controlled vocabulary exist only as candidates ([TAX-03](architecture/domain.md#tax-03)). **Method:** SQL check against [DM-14](architecture/domain.md#dm-14).
- <a id="sc-04-4"></a>**SC-04.4** Parties, dates and case type. **Target:** party extraction F1 ≥ 0.75 (name plus role from [TAX-05](architecture/domain.md#tax-05)), key-date precision ≥ 0.85, and case type suggestion top-1 ≥ 75%. **Method:** harness over gold annotations.
- <a id="sc-04-5"></a>**SC-04.5** Summary faithfulness. **Target:** ≥ 90% of summaries contain no claim that the source does not support. **Method:** lawyer review of 50 sampled summaries against the source text.
- <a id="sc-04-6"></a>**SC-04.6** **(hard)** Provenance completeness. **Target:** 100% of enrichment fields carry the provenance of [DM-11](architecture/domain.md#dm-11) (method, model, prompt version per [LLM-04](architecture/ingestion.md#llm-04), confidence). **Method:** SQL null check over all enriched documents.
- <a id="sc-04-7"></a>**SC-04.7** **(hard)** Overrides survive reprocessing. **Target:** override fields on 20 documents, then re-enrich them with a bumped prompt version through a stage-2 enrich run ([CMD-25](architecture/application.md#cmd-25), [WF-06](architecture/ingestion.md#wf-06)). 100% of overridden fields stay unchanged, and 100% of other fields carry the new prompt version. **Method:** before/after snapshot diff.
- <a id="sc-04-8"></a>**SC-04.8** Structured output reliability. **Target:** ≥ 99% of LLM responses are schema-valid after at most one repair attempt ([LLM-05](architecture/ingestion.md#llm-05)). Documents left with partial or unvalidated enrichment and no review task: 0. **Method:** counters from the enrichment step plus a SQL check.
- <a id="sc-04-9"></a>**SC-04.9** Prompt-injection resistance. **Target:** 0 of the ≥ 20 injection documents get a `doc_type` or `tags` that follow the injected instruction instead of the gold label. **Method:** harness over the injection subset of [SC-15.2](#sc-15-2), with the safeguards of [LLM-06](architecture/ingestion.md#llm-06).

Refs: [F-09](features.md#f-09), [F-10](features.md#f-10), [F-11](features.md#f-11), [F-20](features.md#f-20), [LLM-03](architecture/ingestion.md#llm-03)

<a id="sc-05"></a>
### SC-05 Search relevance

On the evaluation query set, hybrid search finds what lawyers are looking for and beats each of its single legs.

- <a id="sc-05-1"></a>**SC-05.1** Recall. **Target:** hybrid mean recall@10 ≥ 0.80 over [SC-15.3](#sc-15-3), with grade ≥ 2 counted as relevant. **Method:** harness runs [QRY-01](architecture/application.md#qry-01) in hybrid mode for an all-access principal.
- <a id="sc-05-2"></a>**SC-05.2** Hybrid beats single legs. **Target:** hybrid mean nDCG@10 ≥ max(keyword-only, vector-only) + 0.03 absolute, and the 95% paired-bootstrap interval of the difference excludes 0. **Method:** same queries run in the three modes.
- <a id="sc-05-3"></a>**SC-05.3** Exact identifiers are not diluted. **Target:** for exact-identifier queries (matter codes, party names, clause or file numbers), the target document ranks first in ≥ 90% of cases, and hybrid MRR on these queries is ≥ keyword-only MRR. **Method:** per-category metrics over the query set.
- <a id="sc-05-4"></a>**SC-05.4** Filtered recall. **Target:** for filtered queries, recall@10 is within 0.05 of recall@10 over the same relevant set with the filter applied after retrieval, so the pre-filter does not starve the vector leg. **Method:** harness compares pre-filtered and post-filtered runs.
- <a id="sc-05-5"></a>**SC-05.5** Duplicate-free results. **Target:** 0 queries return two members of the same exact [DM-04](architecture/domain.md#dm-04) set in the top 10. **Method:** check over all query results.

Refs: [F-13](features.md#f-13), [F-20](features.md#f-20), [ADR-03](architecture/decisions.md#adr-03)

<a id="sc-06"></a>
### SC-06 Search latency

Interactive queries are fast on the sample corpus.

- <a id="sc-06-1"></a>**SC-06.1** Filtered hybrid search. **Target:** p95 < 1 s and p50 < 300 ms end to end at the API boundary, including the local query-embedding call ([ADP-12.6](architecture/ports-adapters.md#adp-12-6)). The database part alone is p95 < 500 ms. **Method:** replay 1,000 queries drawn from [SC-15.3](#sc-15-3), each with ≥ 2 metadata filters, at concurrency 4 after warm-up. The embedding call share comes from timing spans ([XC-03](architecture/overview.md#xc-03)).
- <a id="sc-06-2"></a>**SC-06.2** Facet counts. **Target:** [QRY-08](architecture/application.md#qry-08) p95 < 1.5 s. **Method:** same replay, with facets requested.
- <a id="sc-06-3"></a>**SC-06.3** Document and thread views. **Target:** [QRY-02](architecture/application.md#qry-02) p95 < 300 ms and [QRY-04](architecture/application.md#qry-04) p95 < 500 ms. **Method:** replay over 500 random visible documents.
- <a id="sc-06-4"></a>**SC-06.4** Latency under ingestion load. **Target:** [SC-06.1](#sc-06-1) p95 < 1.5 s while stage 1 and stage 2 both run under the default GPU-sharing policy on the shared GPU ([WF-04.9](architecture/ingestion.md#wf-04-9)). **Method:** repeat the replay during a run.
- <a id="sc-06-5"></a>**SC-06.5** MCP overhead. **Target:** an MCP tool call adds ≤ 100 ms p95 over the same query through the HTTP API. **Method:** paired replay through both adapters.

Refs: [F-13](features.md#f-13), [F-15](features.md#f-15)

<a id="sc-07"></a>
### SC-07 Scale readiness for 100 GB

Measurement, not hope, shows that the single-store design carries a 100 GB+ corpus.

- <a id="sc-07-1"></a>**SC-07.1** Synthetic scale load. **Target:** with ≥ 10M chunks loaded (halfvec(1024), ≥ 500 clients, ≥ 2,000 matters, realistic metadata skew), filtered hybrid search reaches p95 < 2 s and facet counts p95 < 3 s. **Method:** a generator replicates sample chunks with perturbed vectors and remapped metadata, then the [SC-06.1](#sc-06-1) replay runs against it.
- <a id="sc-07-2"></a>**SC-07.2** Vector index quality at scale. **Target:** the vector leg's recall@10 against exact kNN is ≥ 0.95 on 100 sampled queries, with and without filters. **Method:** compare the HNSW query with an exact scan (index disabled) on the 10M-chunk load.
- <a id="sc-07-3"></a>**SC-07.3** Index build and maintenance. **Target:** HNSW and BM25 build time, peak RAM and on-disk size at 10M chunks are measured and fit the host envelope of [DEP-08](architecture/overview.md#dep-08). **Method:** timed index rebuild with `pg_stat_progress_create_index` and container memory metrics.
- <a id="sc-07-4"></a>**SC-07.4** **(hard)** Capacity model. **Target:** a capacity model, built from measured per-GB ratios (documents, chunks, database bytes per chunk, blob bytes, index RAM, embedding tokens, LLM documents and GPU-hours per stage, stage hours), projects storage, RAM, per-stage duration (stage 1 searchable, stage 2 complete) and model storage for 100 GB. Validation: fitted on half of the sample corpus, the model predicts the full-sample measurements within ±20%. **Method:** the harness exports the ratios, and the model and its assumptions go into the results report ([SC-15.6](#sc-15-6)).
- <a id="sc-07-5"></a>**SC-07.5** No design blocker. **Target:** the 100 GB projection stays below the split-out threshold of [ADR-03](architecture/decisions.md#adr-03) and within [DEP-08](architecture/overview.md#dep-08) sizing, or the report names the component that breaks and its scale lever. **Method:** headroom table in the results report.

Refs: [F-04](features.md#f-04), [F-13](features.md#f-13), [F-19](features.md#f-19), [SCOPE-02](README.md#scope-02), [ADR-12](architecture/decisions.md#adr-12)

<a id="sc-08"></a>
### SC-08 Durable, resumable, idempotent ingestion

Ingestion survives crashes without operator repair, produces the same result as an undisturbed run and does no needless repeated work.

- <a id="sc-08-1"></a>**SC-08.1** **(hard)** Worker crash recovery. **Target:** all worker replicas are SIGKILLed at four points (during discovery paging, mid-OCR, mid-embedding batch and mid-enrichment (stage 2, [WF-07](architecture/ingestion.md#wf-07))). After restart, the ingest run and its companion enrich run resume without operator action within 60 s and complete. **Method:** chaos script against the compose stack. Resumption time comes from workflow status ([WF-05](architecture/ingestion.md#wf-05)).
- <a id="sc-08-2"></a>**SC-08.2** **(hard)** Database restart. **Target:** after a restart of the postgres container mid-run, workers reconnect and the run completes without manual steps. **Method:** chaos script.
- <a id="sc-08-3"></a>**SC-08.3** **(hard)** Same outcome as a control run. **Target:** after [SC-08.1](#sc-08-1) and [SC-08.2](#sc-08-2), counts of documents, chunks, embeddings, enrichments, assignments, review tasks and stage-2 stage statuses equal those of an undisturbed control run over the same scope, and unique-key duplicates are 0. **Method:** SQL comparison of the two databases.
- <a id="sc-08-4"></a>**SC-08.4** Bounded repeated model calls. **Target:** embedding endpoint calls ≤ control-run calls + the batches in flight at each kill, and LLM calls ≤ control-run calls + the in-flight `enrich` items (at most the `LLM_CONCURRENCY` cap of [WF-04.3](architecture/ingestion.md#wf-04-3) per kill). **Method:** call counters in the embedding and LLM adapters.
- <a id="sc-08-5"></a>**SC-08.5** **(hard)** Re-run is a no-op. **Target:** starting a completed run again with the same run key, or a new run over an unchanged source, creates 0 documents, makes 0 LLM calls and 0 embedding calls, and makes 0 stage-2 handoffs (no [WF-07](architecture/ingestion.md#wf-07) enqueued). The new run takes ≤ 5% of the original duration. **Method:** counters and timing ([ING-18](architecture/ingestion.md#ing-18)).
- <a id="sc-08-6"></a>**SC-08.6** Pause, resume and cancel. **Target:** pause, resume and cancel apply to an ingest run and to its companion enrich run independently: pausing one stops its new step starts within 30 s while the other keeps running, resume continues from the checkpoints, and cancel leaves no partially indexed document visible to search. A [CMD-03](architecture/application.md#cmd-03) `throttle` of an enrich run to `max_in_flight` N keeps that run's in-flight `enrich` items ≤ N from its next admission pass on. **Method:** scripted control through [CMD-03](architecture/application.md#cmd-03) with step-start timestamps, in-flight gauges and a search check.
- <a id="sc-08-7"></a>**SC-08.7** Long-run soak. **Target:** one uninterrupted run over the full sample corpus (≥ 8 h) completes without intervention, and worker resident memory grows ≤ 20% after warm-up. **Method:** container memory metrics sampled every minute.
- <a id="sc-08-8"></a>**SC-08.8** Targeted reprocessing. **Target:** bumping only the chunker version reprocesses chunking, embedding and indexing for the affected documents with 0 LLM calls. Bumping only the prompt version makes 0 embedding calls, runs as an enrich run ([WF-06](architecture/ingestion.md#wf-06)) and resets no stage-1 stage. **Method:** stage counters after [CMD-25](architecture/application.md#cmd-25) ([ING-17](architecture/ingestion.md#ing-17)).

Refs: [F-04](features.md#f-04), [F-06](features.md#f-06), [F-22](features.md#f-22), [WF-02](architecture/ingestion.md#wf-02), [WF-06](architecture/ingestion.md#wf-06), [ADR-04](architecture/decisions.md#adr-04)

<a id="sc-09"></a>
### SC-09 Throughput, back-pressure and GPU sharing

Local endpoint capacity and compute caps shape the pace of ingestion but never cause failures. Measured throughput gives a credible 100 GB duration per stage.

- <a id="sc-09-1"></a>**SC-09.1** **(hard)** No back-pressure failures. **Target:** 0 items quarantined because a local model endpoint was saturated, warming up or restarting (class `backpressure`, [XC-05](architecture/overview.md#xc-05)) over the full sample run. The same holds in induced tests: the `llm` and `embed` containers are restarted mid-run, and the fault-injecting endpoint of [ADP-22.10](architecture/ports-adapters.md#adp-22-10) returns 503 for 50% of calls for 10 minutes. **Method:** failure list by error class ([QRY-11](architecture/application.md#qry-11)).
- <a id="sc-09-2"></a>**SC-09.2** Limiter adherence. **Retired:** there are no provider request or token rate limits to adhere to. Replaced by [SC-09.3](#sc-09-3), which checks endpoint concurrency adherence.
- <a id="sc-09-3"></a>**SC-09.3** **(hard)** Local endpoint, LLM and OCR caps. **Target:** concurrent `enrich` requests, concurrent `embed` requests (ingest per the queue cap, query per `embedding.QUERY_CONCURRENCY`) and concurrent OCR jobs never exceed their configured caps ([WF-04](architecture/ingestion.md#wf-04)). **Method:** in-flight gauges ([XC-03.7](architecture/overview.md#xc-03-7)) sampled every 1 s during the soak run of [SC-08.7](#sc-08-7).
- <a id="sc-09-4"></a>**SC-09.4** Bounded backlog. **Target:** queue depth per stage-1 queue stays ≤ the configured enqueue bound of [WF-01](architecture/ingestion.md#wf-01), and open items on the `enrich` queue stay ≤ the admission window of [WF-06.3](architecture/ingestion.md#wf-06-3), for the whole run. **Method:** queue depth sampled every 10 s.
- <a id="sc-09-5"></a>**SC-09.5** Stage throughput. **Target:** items/h, MB/h, chunks/h, LLM documents/h and embedding tokens/min are measured per stage on the reference host. Minimum targets on the reference host: the assumed stage-1 embedding and stage-2 enrichment rates of [DEP-08.6](architecture/overview.md#dep-08-6), sustained, which are estimates and not published benchmarks; GPU ≥ 70% busy during stage-2-bound phases. **Method:** stage counters ([ING-16](architecture/ingestion.md#ing-16)), `nvidia-smi` sampling and vLLM `/metrics` scraping ([XC-03.7](architecture/overview.md#xc-03-7)).
- <a id="sc-09-6"></a>**SC-09.6** Projected 100 GB duration per stage. **Target:** the projection from [SC-07.4](#sc-07-4), with the default vLLM configuration ([DEP-07](architecture/overview.md#dep-07), [DEP-09](architecture/overview.md#dep-09)) on one reference host, is ≤ 7 days from run start for stage 1 (corpus searchable) and ≤ 14 days from stage-1 completion for stage 2 (enrichment complete; [ADR-17.3](architecture/decisions.md#adr-17-3) records why it is not counted from run start), using the projection rule of [DEP-08.6](architecture/overview.md#dep-08-6). The report names the bottleneck per stage and its lever (`enrich` cap, fallback model, second accelerator, `ocr` cap). **Method:** capacity model.

Refs: [F-04](features.md#f-04), [F-22](features.md#f-22), [F-23](features.md#f-23), [XC-05](architecture/overview.md#xc-05), [ADR-15](architecture/decisions.md#adr-15), [ADR-16](architecture/decisions.md#adr-16), [ADR-17](architecture/decisions.md#adr-17)

<a id="sc-10"></a>
### SC-10 Progress observability

An operator can tell at any moment what a long run is doing, how far it has got, when it will finish and what has failed.

- <a id="sc-10-1"></a>**SC-10.1** Same picture on every surface. **Target:** run state, per-stage counts (pending, running, done, failed, quarantined) reported per stage group, throughput and ETA appear in the UI, the CLI and the HTTP API, together with the stage-2 backlog, lag and ETA of the companion enrich run ([ING-16.7](architecture/ingestion.md#ing-16-7)). The three surfaces show identical values for the same instant. **Method:** scripted comparison of the three outputs of [QRY-10](architecture/application.md#qry-10).
- <a id="sc-10-2"></a>**SC-10.2** Freshness. **Target:** displayed counters lag the true state by ≤ 10 s. **Method:** at 20 random points during a run, compare displayed counters with ground-truth SQL counts over [DM-02](architecture/domain.md#dm-02) and record when they converge.
- <a id="sc-10-3"></a>**SC-10.3** ETA accuracy. **Target:** from 25% progress onwards, the ETA is within ±25% of the actual completion time. **Method:** ETA snapshots compared with the completion timestamp.
- <a id="sc-10-4"></a>**SC-10.4** Failure visibility. **Target:** 100% of quarantined items in [SC-01.2](#sc-01-2) are listed with source key, stage, error class, last error message and attempt count. **Method:** compare [QRY-11](architecture/application.md#qry-11) output with the reconciliation.
- <a id="sc-10-5"></a>**SC-10.5** Log correlation. **Target:** 100% of step log lines carry run, workflow and document IDs ([XC-03](architecture/overview.md#xc-03)). For 10 sampled documents, filtering logs by document ID alone reconstructs every processing step. **Method:** JSON log scan plus a manual spot check.

Refs: [F-05](features.md#f-05), [F-21](features.md#f-21)

<a id="sc-11"></a>
### SC-11 Agent access via MCP

An AI agent does real research work through the MCP tools alone, within the principal's permissions.

- <a id="sc-11-1"></a>**SC-11.1** Protocol conformance. **Target:** MCP Inspector lists every tool and resource of [MCP-01](architecture/mcp.md#mcp-01) to [MCP-10](architecture/mcp.md#mcp-10), and every tool accepts schema-valid input and rejects invalid input with an MCP error. **Method:** Inspector session plus a deterministic scripted client.
- <a id="sc-11-2"></a>**SC-11.2** Scripted research tasks. **Target:** a scripted client with fixed tool sequences completes 100% of ≥ 10 tasks. An LLM-driven MCP agent completes ≥ 9 of 10 against a per-task answer rubric. Tasks include: find a matter's engagement letter, read the latest pleading in full, follow a thread from a message and list its attachments, find documents similar to a given one, get oriented on a matter, and flag a misassigned document. **Method:** task suite in the eval tree ([MOD-09](architecture/overview.md#mod-09)), run on the frozen corpus of [SC-15.1](#sc-15-1).
- <a id="sc-11-3"></a>**SC-11.3** Tool coverage and agent writes. **Target:** every tool and resource is used by at least one task. The flag created through [MCP-09](architecture/mcp.md#mcp-09) appears in [QRY-09](architecture/application.md#qry-09) with the agent identity recorded. **Method:** audit and review-task check after the suite.
- <a id="sc-11-4"></a>**SC-11.4** Bounded responses. **Target:** an agent reads a 200-page document completely through chunk-range paging, and no tool response exceeds the configured response-size cap. **Method:** response sizes logged per call.
- <a id="sc-11-5"></a>**SC-11.5** Parity with the HTTP API. **Target:** for the same principal and query, the MCP search returns the same top 10 IDs in the same order as the HTTP API. **Method:** paired run over [SC-15.3](#sc-15-3).
- <a id="sc-11-6"></a>**SC-11.6** **(hard)** Read-only except the flag tool. **Target:** during the whole suite, rows change only in the review-task and audit tables, and only through [MCP-09](architecture/mcp.md#mcp-09) or audit logging. **Method:** table row-count and checksum diff before and after the suite.

Refs: [F-18](features.md#f-18), [ADR-10](architecture/decisions.md#adr-10)

<a id="sc-12"></a>
### SC-12 Permission correctness

Ethical walls and matter grants hold on every read path, and every access is recorded.

- <a id="sc-12-1"></a>**SC-12.1** Wall fixture. **Target:** the fixture holds ≥ 3 walled matters, ≥ 5 principals (including one agent principal, one principal excluded by two walls, one `admin` and one `auditor` who are each excluded by a wall), and canary tokens: unique strings that appear only in walled documents' text, titles and metadata. **Method:** fixture loaded through [CMD-22](architecture/application.md#cmd-22).
- <a id="sc-12-2"></a>**SC-12.2** **(hard)** Zero leaks. **Target:** for every principal and every read surface (search in all three modes, facets, totals, get document, get content, thread, similar, client, matter and person lists, matter overview, every MCP tool and the MCP resource), except [QRY-11](architecture/application.md#qry-11) for `admin` and [QRY-14](architecture/application.md#qry-14) for `auditor`, which are role-gated operational surfaces, 0 responses contain a canary token, a walled document ID or a walled matter name. Requests for a walled ID return the same status and response shape as a nonexistent ID. **Method:** automated leak suite ([XC-07](architecture/overview.md#xc-07)) that enumerates every principal × surface.
- <a id="sc-12-3"></a>**SC-12.3** **(hard)** Counts do not leak. **Target:** facet counts and result totals equal counts computed independently over the documents the principal may see. **Method:** the leak suite recomputes the counts by SQL over the permitted set.
- <a id="sc-12-4"></a>**SC-12.4** Filters cannot widen access. **Target:** a filter that names a walled matter, client or document returns the same empty result as a nonexistent one, on HTTP and MCP. **Method:** leak suite cases.
- <a id="sc-12-5"></a>**SC-12.5** Database backstop. **Target:** a deliberately unfiltered read-model query, run under the read-model database role, returns 0 walled rows. **Method:** test that bypasses the predicate of [XC-01](architecture/overview.md#xc-01).
- <a id="sc-12-6"></a>**SC-12.6** Immediate policy change. **Target:** revoking a grant or adding a wall takes effect on the next query (≤ 1 s), with no reindex. **Method:** leak suite case that changes the policy mid-session.
- <a id="sc-12-7"></a>**SC-12.7** Continuous enforcement. **Target:** the leak suite runs on every change to read-model SQL, query handlers or MCP tools, and the build fails on any leak. **Method:** CI configuration.
- <a id="sc-12-8"></a>**SC-12.8** **(hard)** Audit completeness. **Target:** 100% of queries, MCP calls, overrides and administrative commands issued by the test suites produce an audit entry with the fields of [XC-08](architecture/overview.md#xc-08). **Method:** the test client's call log is reconciled against [QRY-14](architecture/application.md#qry-14) output.

Refs: [F-16](features.md#f-16), [F-17](features.md#f-17), [F-18](features.md#f-18), [ADR-09](architecture/decisions.md#adr-09), [MCP-02](architecture/mcp.md#mcp-02)

<a id="sc-13"></a>
### SC-13 Dedup and threading quality

Each document appears once, near-duplicates are grouped and e-mail conversations are reconstructed.

- <a id="sc-13-1"></a>**SC-13.1** **(hard)** Exact duplicates collapsed. **Target:** 100% of planted exact duplicates in [SC-15.4](#sc-15-4) collapse to one canonical document, and every duplicate keeps its own source location. The planted cases are the same file on the drive, in SharePoint and as an attachment, and the same e-mail in two mailboxes or PSTs. **Method:** harness check over [DM-04](architecture/domain.md#dm-04).
- <a id="sc-13-2"></a>**SC-13.2** Near duplicates. **Target:** pairwise precision ≥ 95% and recall ≥ 80% on the labelled pairs. **Method:** harness over [SC-15.4](#sc-15-4).
- <a id="sc-13-3"></a>**SC-13.3** Thread reconstruction. **Target:** ≥ 95% of labelled messages are in the correct thread (pairwise F1 ≥ 0.95), including messages without In-Reply-To or References headers. **Method:** harness over [SC-15.4](#sc-15-4).
- <a id="sc-13-4"></a>**SC-13.4** No repeated model work on duplicates. **Target:** 0 LLM calls and 0 embedding calls for exact duplicates after the canonical copy. The bytes and documents removed by dedup are reported. **Method:** adapter call counters keyed by document.

Refs: [F-07](features.md#f-07), [F-14](features.md#f-14)

<a id="sc-14"></a>
### SC-14 Architecture conformance and reproducibility

The code follows the chosen architecture, can be checked mechanically and runs anywhere from the README.

- <a id="sc-14-1"></a>**SC-14.1** **(hard)** Dependency rule. **Target:** the import-linter contracts of [ARCH-03](architecture/overview.md#arch-03) pass in CI with 0 violations. **Method:** CI job.
- <a id="sc-14-2"></a>**SC-14.2** Domain and application tests need no I/O. **Target:** domain and application tests pass with in-memory fakes ([ADP-22](architecture/ports-adapters.md#adp-22)) with network and database access blocked, in < 60 s. **Method:** pytest with socket access disabled.
- <a id="sc-14-3"></a>**SC-14.3** Port contracts. **Target:** every outbound port's contract suite passes against both its fake and its PoC adapter. **Method:** parametrised contract tests ([XC-07](architecture/overview.md#xc-07)).
- <a id="sc-14-4"></a>**SC-14.4** Command/query discipline. **Target:** all query handlers pass while running in read-only transactions, and all command handlers return only IDs or status, as [ARCH-04](architecture/overview.md#arch-04) requires. **Method:** architecture test that runs query handlers with `SET TRANSACTION READ ONLY` and checks command handler return types.
- <a id="sc-14-5"></a>**SC-14.5** Adapter swap by configuration. **Target:** switching the LLM runtime from vLLM to llama.cpp server or Ollama ([DEP-07.6](architecture/overview.md#dep-07-6)) needs only configuration changes. So does switching the embedding model from Qwen3-Embedding-4B to BGE-M3 (1024 dimensions, [DM-05.2](architecture/domain.md#dm-05-2), [ADR-16](architecture/decisions.md#adr-16)). The contract tests and a 100-document re-embed and re-enrich pass. **Method:** profile switch plus test run, and a git diff limited to configuration.
- <a id="sc-14-6"></a>**SC-14.6** **(hard)** Zero runtime egress. **Target:** from `docker compose up` (with the `setup` profile down) through a full stage-1 and stage-2 run and the MCP suite of [SC-11.2](#sc-11-2), 0 outbound connections and 0 external DNS resolutions leave any container. The egress self-test ([XC-02.10](architecture/overview.md#xc-02-10)) reports blocked. **Method:** the host nftables log of [DEP-01.9](architecture/overview.md#dep-01-9) shows 0 accepted outbound flows, and a probe from every container to an external IP and to an external name fails ([XC-02](architecture/overview.md#xc-02)). Image pulls and `model-pull` ([DEP-10.2](architecture/overview.md#dep-10-2)) are setup-time and excluded.
- <a id="sc-14-7"></a>**SC-14.7** Fresh-machine reproduction. **Target:** on a clean machine that meets [DEP-08](architecture/overview.md#dep-08), a person outside the build team runs `docker compose up` and one CLI command ([ADP-03](architecture/ports-adapters.md#adp-03)), following only the README. The seeded demo is searchable within 30 min, excluding the image and model downloads of the one-off provisioning of [DEP-10](architecture/overview.md#dep-10). **Method:** timed dry run, with findings recorded.

Refs: [F-19](features.md#f-19), [F-23](features.md#f-23), [ADR-01](architecture/decisions.md#adr-01), [ADR-15](architecture/decisions.md#adr-15)

<a id="sc-15"></a>
### SC-15 Evaluation baseline and verdict

Every SC measurement is reproducible on frozen datasets and stated conditions, and one rule turns the results into a PoC verdict.

- <a id="sc-15-1"></a>**SC-15.1** Sample corpus. The corpus is frozen and holds 10–20 GB and ≥ 50,000 items from firm data covered by the DPIA of [SCOPE-04.6](README.md#scope-04-6), or synthetic, public or anonymised data. It has ≥ 2 PST archives (≥ 20,000 messages in total), ≥ 500 EML, ≥ 500 MSG, ≥ 1,000 OOXML, ≥ 200 legacy Office, ≥ 1,000 born-digital PDF, ≥ 300 scanned PDF or image items, one SharePoint site with ≥ 2 libraries (≥ 1,000 files), and a local-drive tree of ≥ 5,000 files. A deliberate set of ≥ 50 corrupt, encrypted or zero-byte files is included. A manifest records sha256, format class, source and expected outcome for each item.
- <a id="sc-15-2"></a>**SC-15.2** Gold labelled set. The set holds ≥ 500 documents, stratified by format class and covering ≥ 5 clients, ≥ 10 matters and ≥ 4 case types. Firm staff label client, matter, handler, case type, document type, parties with roles and key dates. A 10% subset is labelled twice. An injection subset of ≥ 20 documents carries instructions in body text, white or hidden text, metadata fields and attachment names, labelled with their true document type and tags ([SC-04.9](#sc-04-9)). Inter-annotator agreement (Cohen's kappa) is reported per field, and a field with kappa < 0.7 is reported without being judged against its target.
- <a id="sc-15-3"></a>**SC-15.3** Evaluation query set. It holds ≥ 60 queries written by lawyers: ≥ 20 exact-identifier queries, ≥ 20 conceptual natural-language queries and ≥ 20 with metadata filters. Relevance is graded 0–3 over the pooled top 20 of the hybrid, keyword-only and vector-only modes.
- <a id="sc-15-4"></a>**SC-15.4** Dedup and thread gold. It holds ≥ 100 planted exact-duplicate groups across sources, ≥ 200 labelled near-duplicate and non-duplicate pairs, and ≥ 50 labelled threads (≥ 400 messages), including forwarded and header-stripped messages.
- <a id="sc-15-5"></a>**SC-15.5** Reference conditions. Measurements run on a host that meets [DEP-08](architecture/overview.md#dep-08). The host specification (including the accelerator model and its memory), the LLM and embedding model names with their weights digests from `models.lock` ([DEP-10.1](architecture/overview.md#dep-10-1)), the runtime image digests, `--gpu-memory-utilization` and `--max-num-seqs` per model endpoint, the MCP protocol revision and SDK version ([MCP-01.3](architecture/mcp.md#mcp-01-3)), and all version stamps ([XC-06](architecture/overview.md#xc-06)) are recorded with each result.
- <a id="sc-15-6"></a>**SC-15.6** **(hard)** Results report. The evaluation harness ([F-20](features.md#f-20)) writes one versioned report per evaluation run. For each check ID it lists the measured value, target, pass or fail, and the dataset and version hashes. The report also holds the capacity model of [SC-07.4](#sc-07-4). Every check in this file appears in the report.
- <a id="sc-15-7"></a>**SC-15.7** Verdict rule. The PoC succeeds when every **(hard)** check passes and ≥ 80% of the other checks pass. Every missed check needs a root cause and a remediation in the report. The PoC fails if any hard check fails.

Refs: [F-20](features.md#f-20), [MOD-09](architecture/overview.md#mod-09), [SCOPE-02](README.md#scope-02)

<a id="sc-16"></a>
### SC-16 Local inference and two-stage ingestion

Documents become searchable without the LLM. Stage 2 runs, pauses and re-runs on its own. Both models fit the 96 GB budget.

- <a id="sc-16-1"></a>**SC-16.1** **(hard)** Searchable without LLM. **Target:** a full-sample run with `enrich_mode=none` and the `llm` service stopped reaches completed. 100% of non-quarantined canonical documents with text are indexed, with vectors, rule-derived assignments and access keys, 0 LLM calls are made, and 0 accepted stage-1 assignments for new text carry `llm` provenance ([ING-11.1](architecture/ingestion.md#ing-11-1)). Hybrid recall@10 over the [SC-15.3](#sc-15-3) queries that use no enrichment-field filter is within 0.05 of the post-stage-2 value. **Method:** harness, LLM adapter call counters and SQL ([ING-21](architecture/ingestion.md#ing-21)).
- <a id="sc-16-2"></a>**SC-16.2** **(hard)** Stage 2 re-runs independently. **Target:** a prompt-version bump re-enriches ≥ 1,000 documents through an enrich run ([WF-06](architecture/ingestion.md#wf-06)). During the run: 0 chunk rows inserted, updated or deleted (`pg_stat_user_tables` counters for `chunks`), no growth of the HNSW and BM25 index relations, 0 vectors written and 0 stage-1 stage transitions ([ADP-09.5](architecture/ports-adapters.md#adp-09-5)); every probe query issued every 10 s returns the same visible document set; overrides are untouched ([SC-04.7](#sc-04-7)). **Method:** SQL row checksums before and after, plus the probe log.
- <a id="sc-16-3"></a>**SC-16.3** Stage-2 pause and lag. **Target:** with stage 2 paused, a 10,000-item stage-1 scope completes with throughput within 10% of a stage-1-only control. After resume, every eligible document is enriched exactly once (enrichment count by idempotency key = eligible documents), and lag is shown within the 10 s freshness of [SC-10.2](#sc-10-2). **Method:** run counters ([ING-16.7](architecture/ingestion.md#ing-16-7)) and SQL.
- <a id="sc-16-4"></a>**SC-16.4** **(hard)** Model fit within 96 GB. **Target:** with the default models and the configured memory fractions ([DEP-08.8](architecture/overview.md#dep-08-8)), peak device memory of `llm` + `embed` is ≤ 96 GB during the soak at full concurrency with maximum-length prompts. 0 out-of-memory errors and 0 endpoint restarts. The KV capacity that vLLM reports is ≥ 16 concurrent sequences × 16k tokens. Preemption counts are reported. **Method:** `nvidia-smi` sampled every 1 s plus vLLM `/metrics` ([XC-03.7](architecture/overview.md#xc-03-7)).
- <a id="sc-16-5"></a>**SC-16.5** Pinning and warm-up. **Target:** 100% of enrichment provenance and chunk stamps carry the weights digests in `models.lock` ([DEP-10.1](architecture/overview.md#dep-10-1)). A single altered weight file makes `model-verify` fail and the model services refuse to start ([DEP-10.3](architecture/overview.md#dep-10-3)). A cold start makes no runtime download attempt for weight, tokenizer or vocab artefacts, including the Harmony and tiktoken encodings of [DEP-07.1](architecture/overview.md#dep-07-1) (0 entries in the nftables drop log, [DEP-01.9](architecture/overview.md#dep-01-9)), and produces 0 step failures, because lanes admit work only after readiness and warm-up ([ADP-12.7](architecture/ports-adapters.md#adp-12-7), [ADP-13.4](architecture/ports-adapters.md#adp-13-4)). **Method:** tamper test, SQL and a cold-start run.
- <a id="sc-16-6"></a>**SC-16.6** GPU sharing priority. **Target:** while an enrich run with a backlog runs under the default GPU-sharing policy ([WF-04.9](architecture/ingestion.md#wf-04-9)), stage-1 embedding throughput is ≥ 70% of its stage-1-only rate, and [SC-06.4](#sc-06-4) holds. A control run with `queues.GPU_SHARED=false` reports the unshared figure. **Method:** paired runs; window size and embedding rate come from [ING-16](architecture/ingestion.md#ing-16) and [XC-03.7](architecture/overview.md#xc-03-7).

Refs: [F-22](features.md#f-22), [F-23](features.md#f-23), [ING-21](architecture/ingestion.md#ing-21), [ADR-15](architecture/decisions.md#adr-15), [ADR-16](architecture/decisions.md#adr-16), [ADR-17](architecture/decisions.md#adr-17)
