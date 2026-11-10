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
| [SC-09](#sc-09) | Throughput, rate limiting and throttling | [SC-09.1](#sc-09-1), [SC-09.3](#sc-09-3) |
| [SC-10](#sc-10) | Progress observability | none |
| [SC-11](#sc-11) | Agent access via MCP | [SC-11.6](#sc-11-6) |
| [SC-12](#sc-12) | Permission correctness | [SC-12.2](#sc-12-2), [SC-12.3](#sc-12-3), [SC-12.8](#sc-12-8) |
| [SC-13](#sc-13) | Dedup and threading quality | [SC-13.1](#sc-13-1) |
| [SC-14](#sc-14) | Architecture conformance and reproducibility | [SC-14.1](#sc-14-1), [SC-14.6](#sc-14-6) |
| [SC-15](#sc-15) | Evaluation baseline and verdict | [SC-15.6](#sc-15-6) |

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

- <a id="sc-03-1"></a>**SC-03.1** Client and matter accuracy. **Target:** ≥ 90% each on the gold set ([SC-15.2](#sc-15-2)), counting the highest-ranked assignment after automated processing and before any human review. **Method:** harness compares [DM-10](architecture/domain.md#dm-10) assignments to gold labels. Documents with no proposal count as wrong.
- <a id="sc-03-2"></a>**SC-03.2** Auto-accept precision. **Target:** ≥ 97% precision for client and matter assignments that were auto-accepted under the confidence bands of [TAX-04](architecture/domain.md#tax-04). **Method:** precision over the auto-accepted subset of the gold set.
- <a id="sc-03-3"></a>**SC-03.3** Review load. **Target:** ≥ 60% of gold documents are auto-accepted for both client and matter, so that ≤ 40% need review. **Method:** share of gold documents by assignment status.
- <a id="sc-03-4"></a>**SC-03.4** **(hard)** Routing completeness. **Target:** 100% of documents that have no auto-accepted client and matter, or conflicting rule and LLM signals, have an open review task ([DM-13](architecture/domain.md#dm-13)), including documents with no matter candidate at all ([DM-10.6](architecture/domain.md#dm-10-6) rule 6). Unassigned documents without a task: 0. **Method:** SQL anti-join over the full sample corpus.
- <a id="sc-03-5"></a>**SC-03.5** Handler and case type accuracy. **Target:** ≥ 85% each on the gold set. **Method:** same as [SC-03.1](#sc-03-1).
- <a id="sc-03-6"></a>**SC-03.6** Family and thread consistency. **Target:** 100% of attachments and thread members without a human override carry the same client and matter as their family parent or thread, as inheritance rules require. A later matter change reaches them through the durable reprocess run requested by the [EVT-04](architecture/domain.md#evt-04) handler ([WF-03.6](architecture/ingestion.md#wf-03-6)); the check runs after those runs complete. **Method:** SQL consistency check.
- <a id="sc-03-7"></a>**SC-03.7** Override takes effect. **Target:** after a reviewer resolves a task in the UI, the very next search filtered by the new matter returns the document, and its provenance shows method human. Median reviewer time per task is ≤ 30 s over 50 tasks. **Method:** scripted UI session timing plus a [QRY-01](architecture/application.md#qry-01) check after each resolution.

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
- <a id="sc-04-7"></a>**SC-04.7** **(hard)** Overrides survive reprocessing. **Target:** override fields on 20 documents, then re-enrich them with a bumped prompt version through [WF-03](architecture/ingestion.md#wf-03). 100% of overridden fields stay unchanged, and 100% of other fields carry the new prompt version. **Method:** before/after snapshot diff.
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

- <a id="sc-06-1"></a>**SC-06.1** Filtered hybrid search. **Target:** p95 < 1 s and p50 < 300 ms end to end at the API boundary, including the query-embedding API call. The database part alone is p95 < 500 ms. **Method:** replay 1,000 queries drawn from [SC-15.3](#sc-15-3), each with ≥ 2 metadata filters, at concurrency 4 after warm-up. The embedding call share comes from timing spans ([XC-03](architecture/overview.md#xc-03)).
- <a id="sc-06-2"></a>**SC-06.2** Facet counts. **Target:** [QRY-08](architecture/application.md#qry-08) p95 < 1.5 s. **Method:** same replay, with facets requested.
- <a id="sc-06-3"></a>**SC-06.3** Document and thread views. **Target:** [QRY-02](architecture/application.md#qry-02) p95 < 300 ms and [QRY-04](architecture/application.md#qry-04) p95 < 500 ms. **Method:** replay over 500 random visible documents.
- <a id="sc-06-4"></a>**SC-06.4** Latency under ingestion load. **Target:** [SC-06.1](#sc-06-1) p95 < 1.5 s while an ingestion run is at full configured concurrency. **Method:** repeat the replay during a run.
- <a id="sc-06-5"></a>**SC-06.5** MCP overhead. **Target:** an MCP tool call adds ≤ 100 ms p95 over the same query through the HTTP API. **Method:** paired replay through both adapters.

Refs: [F-13](features.md#f-13), [F-15](features.md#f-15)

<a id="sc-07"></a>
### SC-07 Scale readiness for 100 GB

Measurement, not hope, shows that the single-store design carries a 100 GB+ corpus.

- <a id="sc-07-1"></a>**SC-07.1** Synthetic scale load. **Target:** with ≥ 10M chunks loaded (halfvec(1024), ≥ 500 clients, ≥ 2,000 matters, realistic metadata skew), filtered hybrid search reaches p95 < 2 s and facet counts p95 < 3 s. **Method:** a generator replicates sample chunks with perturbed vectors and remapped metadata, then the [SC-06.1](#sc-06-1) replay runs against it.
- <a id="sc-07-2"></a>**SC-07.2** Vector index quality at scale. **Target:** the vector leg's recall@10 against exact kNN is ≥ 0.95 on 100 sampled queries, with and without filters. **Method:** compare the HNSW query with an exact scan (index disabled) on the 10M-chunk load.
- <a id="sc-07-3"></a>**SC-07.3** Index build and maintenance. **Target:** HNSW and BM25 build time, peak RAM and on-disk size at 10M chunks are measured and fit the host envelope of [DEP-08](architecture/overview.md#dep-08). **Method:** timed index rebuild with `pg_stat_progress_create_index` and container memory metrics.
- <a id="sc-07-4"></a>**SC-07.4** **(hard)** Capacity model. **Target:** a capacity model, built from measured per-GB ratios (documents, chunks, database bytes per chunk, blob bytes, index RAM, embedding tokens and cost, stage hours), projects storage, RAM, ingestion duration and embedding cost for 100 GB. Validation: fitted on half of the sample corpus, the model predicts the full-sample measurements within ±20%. **Method:** the harness exports the ratios, and the model and its assumptions go into the results report ([SC-15.6](#sc-15-6)).
- <a id="sc-07-5"></a>**SC-07.5** No design blocker. **Target:** the 100 GB projection stays below the split-out threshold of [ADR-03](architecture/decisions.md#adr-03) and within [DEP-08](architecture/overview.md#dep-08) sizing, or the report names the component that breaks and its scale lever. **Method:** headroom table in the results report.

Refs: [F-04](features.md#f-04), [F-13](features.md#f-13), [F-19](features.md#f-19), [SCOPE-02](README.md#scope-02), [ADR-12](architecture/decisions.md#adr-12)

<a id="sc-08"></a>
### SC-08 Durable, resumable, idempotent ingestion

Ingestion survives crashes without operator repair, produces the same result as an undisturbed run and does no needless repeated work.

- <a id="sc-08-1"></a>**SC-08.1** **(hard)** Worker crash recovery. **Target:** all worker replicas are SIGKILLed at three points (during discovery paging, mid-OCR and mid-embedding batch). After restart, the run resumes without operator action within 60 s and completes. **Method:** chaos script against the compose stack. Resumption time comes from workflow status ([WF-05](architecture/ingestion.md#wf-05)).
- <a id="sc-08-2"></a>**SC-08.2** **(hard)** Database restart. **Target:** after a restart of the postgres container mid-run, workers reconnect and the run completes without manual steps. **Method:** chaos script.
- <a id="sc-08-3"></a>**SC-08.3** **(hard)** Same outcome as a control run. **Target:** after [SC-08.1](#sc-08-1) and [SC-08.2](#sc-08-2), counts of documents, chunks, embeddings, assignments and review tasks equal those of an undisturbed control run over the same scope, and unique-key duplicates are 0. **Method:** SQL comparison of the two databases.
- <a id="sc-08-4"></a>**SC-08.4** Bounded repeated external calls. **Target:** embedding API calls ≤ control-run calls + the batches in flight at each kill, and LLM calls ≤ control-run calls + the in-flight items (at most the [WF-04](architecture/ingestion.md#wf-04) cap per kill). **Method:** call counters in the embedding and LLM adapters.
- <a id="sc-08-5"></a>**SC-08.5** **(hard)** Re-run is a no-op. **Target:** starting a completed run again with the same run key, or a new run over an unchanged source, creates 0 documents and makes 0 LLM calls and 0 embedding calls. The new run takes ≤ 5% of the original duration. **Method:** counters and timing ([ING-18](architecture/ingestion.md#ing-18)).
- <a id="sc-08-6"></a>**SC-08.6** Pause, resume and cancel. **Target:** pause stops new step starts within 30 s, resume continues from the checkpoints, and cancel leaves no partially indexed document visible to search. **Method:** scripted control through [CMD-03](architecture/application.md#cmd-03) with step-start timestamps and a search check.
- <a id="sc-08-7"></a>**SC-08.7** Long-run soak. **Target:** one uninterrupted run over the full sample corpus (≥ 8 h) completes without intervention, and worker resident memory grows ≤ 20% after warm-up. **Method:** container memory metrics sampled every minute.
- <a id="sc-08-8"></a>**SC-08.8** Targeted reprocessing. **Target:** bumping only the chunker version reprocesses chunking, embedding and indexing for the affected documents with 0 LLM calls. Bumping only the prompt version makes 0 embedding calls. **Method:** stage counters after [CMD-25](architecture/application.md#cmd-25) ([ING-17](architecture/ingestion.md#ing-17)).

Refs: [F-04](features.md#f-04), [F-06](features.md#f-06), [WF-02](architecture/ingestion.md#wf-02), [ADR-04](architecture/decisions.md#adr-04)

<a id="sc-09"></a>
### SC-09 Throughput, rate limiting and throttling

External rate limits and local compute limits shape the pace of ingestion but never cause failures. Measured throughput gives a credible 100 GB duration.

- <a id="sc-09-1"></a>**SC-09.1** **(hard)** No rate-limit failures. **Target:** 0 items quarantined because of embedding-API 429 or quota responses over the full sample run. The same holds in an induced test where a fault-injecting provider fake ([ADP-22](architecture/ports-adapters.md#adp-22)) returns 429 for 50% of calls for 10 minutes. **Method:** failure list by error class ([QRY-11](architecture/application.md#qry-11)).
- <a id="sc-09-2"></a>**SC-09.2** Limiter adherence. **Target:** the embedding request and token rates never exceed the configured limits in any 60 s window. **Method:** per-request timestamps and token counts from the embedding adapter.
- <a id="sc-09-3"></a>**SC-09.3** **(hard)** Local LLM and OCR caps. **Target:** concurrent LLM requests and concurrent OCR jobs never exceed their configured caps ([WF-04](architecture/ingestion.md#wf-04)). **Method:** in-flight gauges sampled every 1 s during the soak run of [SC-08.7](#sc-08-7).
- <a id="sc-09-4"></a>**SC-09.4** Bounded backlog. **Target:** queue depth per queue stays ≤ the configured enqueue bound of [WF-01](architecture/ingestion.md#wf-01) for the whole run. **Method:** queue depth sampled every 10 s.
- <a id="sc-09-5"></a>**SC-09.5** Stage throughput. **Target:** items/h, MB/h, chunks/h, LLM documents/h and embedding tokens/min are measured per stage on the reference host, and the GPU is ≥ 70% busy during LLM-bound phases. **Method:** stage counters ([ING-16](architecture/ingestion.md#ing-16)) plus `nvidia-smi` sampling.
- <a id="sc-09-6"></a>**SC-09.6** Projected 100 GB duration. **Target:** the projection from [SC-07.4](#sc-07-4), with the `vllm` profile of [DEP-07.4](architecture/overview.md#dep-07-4), is ≤ 14 days of wall-clock time on one reference host. The default Ollama configuration is expected to miss 14 days ([DEP-08.6](architecture/overview.md#dep-08-6)); the report states its projection too but it does not count against this target. The report names the bottleneck stage and its scale lever (more GPU, smaller model, more workers). **Method:** capacity model.

Refs: [F-04](features.md#f-04), [XC-05](architecture/overview.md#xc-05), [ADR-06](architecture/decisions.md#adr-06)

<a id="sc-10"></a>
### SC-10 Progress observability

An operator can tell at any moment what a long run is doing, how far it has got, when it will finish and what has failed.

- <a id="sc-10-1"></a>**SC-10.1** Same picture on every surface. **Target:** run state, per-stage counts (pending, running, done, failed, quarantined), throughput and ETA appear in the UI, the CLI and the HTTP API, and the three surfaces show identical values for the same instant. **Method:** scripted comparison of the three outputs of [QRY-10](architecture/application.md#qry-10).
- <a id="sc-10-2"></a>**SC-10.2** Freshness. **Target:** displayed counters lag the true state by ≤ 10 s. **Method:** at 20 random points during a run, compare displayed counters with ground-truth SQL counts over [DM-02](architecture/domain.md#dm-02) and record when they converge.
- <a id="sc-10-3"></a>**SC-10.3** ETA accuracy. **Target:** from 25% progress onwards, the ETA is within ±25% of the actual completion time. **Method:** ETA snapshots compared with the completion timestamp.
- <a id="sc-10-4"></a>**SC-10.4** Failure visibility. **Target:** 100% of quarantined items in [SC-01.2](#sc-01-2) are listed with source key, stage, error class, last error message and attempt count. **Method:** compare [QRY-11](architecture/application.md#qry-11) output with the reconciliation.
- <a id="sc-10-5"></a>**SC-10.5** Log correlation. **Target:** 100% of step log lines carry run, workflow and document IDs ([XC-03](architecture/overview.md#xc-03)). For 10 sampled documents, filtering logs by document ID alone reconstructs every processing step. **Method:** JSON log scan plus a manual spot check.

Refs: [F-05](features.md#f-05), [F-21](features.md#f-21)

<a id="sc-11"></a>
### SC-11 Agent access via MCP

An AI agent does real research work through the MCP tools alone, within the principal's permissions.

- <a id="sc-11-1"></a>**SC-11.1** Protocol conformance. **Target:** MCP Inspector lists every tool and resource of [MCP-01](architecture/mcp.md#mcp-01) to [MCP-10](architecture/mcp.md#mcp-10), and every tool accepts schema-valid input and rejects invalid input with an MCP error. **Method:** Inspector session plus a deterministic scripted client.
- <a id="sc-11-2"></a>**SC-11.2** Scripted research tasks. **Target:** a scripted client with fixed tool sequences completes 100% of ≥ 10 tasks. An LLM-driven MCP agent completes ≥ 9 of 10 against a per-task answer rubric. Tasks include: find a matter's engagement letter, read the latest pleading in full, follow a thread from a message and list its attachments, find documents similar to a given one, get oriented on a matter, and flag a misassigned document. **Method:** task suite in the eval tree ([MOD-09](architecture/overview.md#mod-09)), run on sample or consented data only ([SCOPE-04](README.md#scope-04)).
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
- <a id="sc-13-4"></a>**SC-13.4** No paid work on duplicates. **Target:** 0 LLM calls and 0 embedding calls for exact duplicates after the canonical copy. The bytes and documents removed by dedup are reported. **Method:** adapter call counters keyed by document.

Refs: [F-07](features.md#f-07), [F-14](features.md#f-14)

<a id="sc-14"></a>
### SC-14 Architecture conformance and reproducibility

The code follows the chosen architecture, can be checked mechanically and runs anywhere from the README.

- <a id="sc-14-1"></a>**SC-14.1** **(hard)** Dependency rule. **Target:** the import-linter contracts of [ARCH-03](architecture/overview.md#arch-03) pass in CI with 0 violations. **Method:** CI job.
- <a id="sc-14-2"></a>**SC-14.2** Domain and application tests need no I/O. **Target:** domain and application tests pass with in-memory fakes ([ADP-22](architecture/ports-adapters.md#adp-22)) with network and database access blocked, in < 60 s. **Method:** pytest with socket access disabled.
- <a id="sc-14-3"></a>**SC-14.3** Port contracts. **Target:** every outbound port's contract suite passes against both its fake and its PoC adapter. **Method:** parametrised contract tests ([XC-07](architecture/overview.md#xc-07)).
- <a id="sc-14-4"></a>**SC-14.4** Command/query discipline. **Target:** all query handlers pass while running in read-only transactions, and all command handlers return only IDs or status, as [ARCH-04](architecture/overview.md#arch-04) requires. **Method:** architecture test that runs query handlers with `SET TRANSACTION READ ONLY` and checks command handler return types.
- <a id="sc-14-5"></a>**SC-14.5** Adapter swap by configuration. **Target:** switching the embedding provider from [ADP-11](architecture/ports-adapters.md#adp-11) to [ADP-12](architecture/ports-adapters.md#adp-12), and the LLM runtime from Ollama to vLLM, needs only configuration changes. The target embedding model has 1024 dimensions ([DM-05.2](architecture/domain.md#dm-05-2)), and the egress allowlist change of [DEP-01.4](architecture/overview.md#dep-01-4) counts as configuration. The contract tests and a 100-document re-embed and re-enrich pass. **Method:** profile switch plus test run, and a git diff limited to configuration.
- <a id="sc-14-6"></a>**SC-14.6** **(hard)** Egress conformance. **Target:** during a run, no container can reach any external host except the embedding API host, and LLM traffic goes only to the local LLM service. **Method:** connection attempts from each container to an arbitrary external host fail, and proxy or firewall logs during a full run show only the embedding host ([DEP-01](architecture/overview.md#dep-01), [XC-02](architecture/overview.md#xc-02)). The `setup` profile and image pulls are excluded: `ollama-pull` is stopped before data load ([DEP-01.7](architecture/overview.md#dep-01-7)).
- <a id="sc-14-7"></a>**SC-14.7** Fresh-machine reproduction. **Target:** on a clean machine that meets [DEP-08](architecture/overview.md#dep-08), a person outside the build team runs `docker compose up` and one CLI command ([ADP-03](architecture/ports-adapters.md#adp-03)), following only the README. The seeded demo is searchable within 30 min, excluding image and model downloads. **Method:** timed dry run, with findings recorded.

Refs: [F-19](features.md#f-19), [ADR-01](architecture/decisions.md#adr-01)

<a id="sc-15"></a>
### SC-15 Evaluation baseline and verdict

Every SC measurement is reproducible on frozen datasets and stated conditions, and one rule turns the results into a PoC verdict.

- <a id="sc-15-1"></a>**SC-15.1** Sample corpus. The corpus is frozen and holds 10–20 GB and ≥ 50,000 items from sample or consented data only ([SCOPE-04](README.md#scope-04)). It has ≥ 2 PST archives (≥ 20,000 messages in total), ≥ 500 EML, ≥ 500 MSG, ≥ 1,000 OOXML, ≥ 200 legacy Office, ≥ 1,000 born-digital PDF, ≥ 300 scanned PDF or image items, one SharePoint site with ≥ 2 libraries (≥ 1,000 files), and a local-drive tree of ≥ 5,000 files. A deliberate set of ≥ 50 corrupt, encrypted or zero-byte files is included. A manifest records sha256, format class, source and expected outcome for each item.
- <a id="sc-15-2"></a>**SC-15.2** Gold labelled set. The set holds ≥ 500 documents, stratified by format class and covering ≥ 5 clients, ≥ 10 matters and ≥ 4 case types. Firm staff label client, matter, handler, case type, document type, parties with roles and key dates. A 10% subset is labelled twice. An injection subset of ≥ 20 documents carries instructions in body text, white or hidden text, metadata fields and attachment names, labelled with their true document type and tags ([SC-04.9](#sc-04-9)). Inter-annotator agreement (Cohen's kappa) is reported per field, and a field with kappa < 0.7 is reported without being judged against its target.
- <a id="sc-15-3"></a>**SC-15.3** Evaluation query set. It holds ≥ 60 queries written by lawyers: ≥ 20 exact-identifier queries, ≥ 20 conceptual natural-language queries and ≥ 20 with metadata filters. Relevance is graded 0–3 over the pooled top 20 of the hybrid, keyword-only and vector-only modes.
- <a id="sc-15-4"></a>**SC-15.4** Dedup and thread gold. It holds ≥ 100 planted exact-duplicate groups across sources, ≥ 200 labelled near-duplicate and non-duplicate pairs, and ≥ 50 labelled threads (≥ 400 messages), including forwarded and header-stripped messages.
- <a id="sc-15-5"></a>**SC-15.5** Reference conditions. Measurements run on a host that meets [DEP-08](architecture/overview.md#dep-08). The host specification, the pinned model and embedding identifiers, the MCP protocol revision and SDK version ([MCP-01.3](architecture/mcp.md#mcp-01-3)), and all version stamps ([XC-06](architecture/overview.md#xc-06)) are recorded with each result.
- <a id="sc-15-6"></a>**SC-15.6** **(hard)** Results report. The evaluation harness ([F-20](features.md#f-20)) writes one versioned report per evaluation run. For each check ID it lists the measured value, target, pass or fail, and the dataset and version hashes. The report also holds the capacity model of [SC-07.4](#sc-07-4). Every check in this file appears in the report.
- <a id="sc-15-7"></a>**SC-15.7** Verdict rule. The PoC succeeds when every **(hard)** check passes and ≥ 80% of the other checks pass. Every missed check needs a root cause and a remediation in the report. The PoC fails if any hard check fails.

Refs: [F-20](features.md#f-20), [MOD-09](architecture/overview.md#mod-09), [SCOPE-02](README.md#scope-02)
