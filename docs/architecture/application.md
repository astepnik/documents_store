# Application layer: commands and queries

This file owns the application API: commands (CMD-), which change aggregates, and queries (QRY-), which read through the read model. It also owns the handler contracts [CMD-26](#cmd-26) and [QRY-16](#qry-16). The split itself is defined in [ARCH-04](overview.md#arch-04) and the transaction rule in [ARCH-06](overview.md#arch-06). Handlers live in `docstore.application` ([MOD-03](overview.md#mod-03)) and are reached through [PORT-01](ports-adapters.md#port-01) and [PORT-02](ports-adapters.md#port-02). Business rules belong to the domain ([domain.md](domain.md)); handlers load, call domain methods and policies, save, and call ports.

Roles in the "Callers" columns are the principal roles of [DM-16.2](domain.md#dm-16-2). "Invoked by" names the inbound adapter or pipeline step that issues the call.

## Commands

| ID | Command | Aggregate | Callers | Invoked by | Events |
|---|---|---|---|---|---|
| [CMD-01](#cmd-01) | RegisterSource | [DM-01](domain.md#dm-01) | admin | [ADP-03](ports-adapters.md#adp-03), [ADP-01](ports-adapters.md#adp-01) | none |
| [CMD-02](#cmd-02) | StartIngestionRun | [DM-17](domain.md#dm-17) (new) | admin | [ADP-03](ports-adapters.md#adp-03), [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02) | [EVT-10](domain.md#evt-10) |
| [CMD-03](#cmd-03) | ControlIngestionRun | [DM-17](domain.md#dm-17) | admin | [ADP-03](ports-adapters.md#adp-03), [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02) | [EVT-10](domain.md#evt-10) |
| [CMD-04](#cmd-04) | DiscoverSourceItems | [DM-02](domain.md#dm-02) (batch), [DM-01](domain.md#dm-01) cursor | worker | [ING-01](ingestion.md#ing-01) | none |
| [CMD-05](#cmd-05) | IngestSourceItem | [DM-03](domain.md#dm-03) | worker | [ING-02](ingestion.md#ing-02), [ING-03](ingestion.md#ing-03) | [EVT-01](domain.md#evt-01), [EVT-03](domain.md#evt-03) |
| [CMD-06](#cmd-06) | ExpandContainer | [DM-02](domain.md#dm-02) (new children) | worker | [ING-04](ingestion.md#ing-04) | none |
| [CMD-07](#cmd-07) | ExtractDocumentText | [DM-03](domain.md#dm-03) | worker | [ING-05](ingestion.md#ing-05), [ING-06](ingestion.md#ing-06) | [EVT-02](domain.md#evt-02), [EVT-03](domain.md#evt-03) |
| [CMD-08](#cmd-08) | LinkFamilyAndThread | [DM-06](domain.md#dm-06) | worker | [ING-07](ingestion.md#ing-07) | none |
| [CMD-09](#cmd-09) | DetectNearDuplicates | [DM-04](domain.md#dm-04) | worker | [ING-08](ingestion.md#ing-08) | [EVT-03](domain.md#evt-03) |
| [CMD-10](#cmd-10) | ApplyAssignmentRules | [DM-03](domain.md#dm-03) | worker | [ING-09](ingestion.md#ing-09) (also in [WF-03](ingestion.md#wf-03) runs), [EVT-03](domain.md#evt-03) handler | [EVT-04](domain.md#evt-04) |
| [CMD-11](#cmd-11) | EnrichDocument | [DM-03](domain.md#dm-03) | worker | [ING-10](ingestion.md#ing-10) | [EVT-05](domain.md#evt-05) |
| [CMD-12](#cmd-12) | ReconcileAssignments | [DM-03](domain.md#dm-03), [DM-13](domain.md#dm-13) (new) | worker | [ING-11](ingestion.md#ing-11) (also in [WF-03](ingestion.md#wf-03) runs) | [EVT-04](domain.md#evt-04), [EVT-07](domain.md#evt-07) |
| [CMD-13](#cmd-13) | ChunkDocument | [DM-03](domain.md#dm-03) | worker | [ING-12](ingestion.md#ing-12) | none |
| [CMD-14](#cmd-14) | EmbedChunks | [DM-03](domain.md#dm-03) | worker | [ING-13](ingestion.md#ing-13) | none |
| [CMD-15](#cmd-15) | IndexDocument | [DM-03](domain.md#dm-03) | worker | [ING-14](ingestion.md#ing-14) | [EVT-09](domain.md#evt-09) |
| [CMD-16](#cmd-16) | RecordStepFailure | [DM-02](domain.md#dm-02), [DM-17](domain.md#dm-17) counters | worker | [ING-15](ingestion.md#ing-15) | none |
| [CMD-17](#cmd-17) | OverrideAssignment | [DM-03](domain.md#dm-03) | reviewer | [ADP-02](ports-adapters.md#adp-02), [ADP-01](ports-adapters.md#adp-01), [CMD-19](#cmd-19) | [EVT-04](domain.md#evt-04), [EVT-06](domain.md#evt-06) |
| [CMD-18](#cmd-18) | OverrideEnrichment | [DM-03](domain.md#dm-03) | reviewer | [ADP-02](ports-adapters.md#adp-02), [ADP-01](ports-adapters.md#adp-01), [CMD-19](#cmd-19) | [EVT-06](domain.md#evt-06) |
| [CMD-19](#cmd-19) | ResolveReviewTask | [DM-13](domain.md#dm-13) | reviewer; system for `superseded` | [ADP-02](ports-adapters.md#adp-02), [ADP-01](ports-adapters.md#adp-01), [EVT-06](domain.md#evt-06) handler | [EVT-08](domain.md#evt-08) |
| [CMD-20](#cmd-20) | FlagForReview | [DM-13](domain.md#dm-13) | member, reviewer, agent | [ADP-02](ports-adapters.md#adp-02), [ADP-01](ports-adapters.md#adp-01), [MCP-09](mcp.md#mcp-09) | [EVT-07](domain.md#evt-07) |
| [CMD-21](#cmd-21) | ImportReferenceData | [DM-07](domain.md#dm-07), [DM-08](domain.md#dm-08), [DM-09](domain.md#dm-09) (batch) | admin | [ADP-03](ports-adapters.md#adp-03) | none |
| [CMD-22](#cmd-22) | ImportAccessPolicies | [DM-16](domain.md#dm-16) (batch) | admin | [ADP-03](ports-adapters.md#adp-03) | none |
| [CMD-23](#cmd-23) | UpsertTaxonomyTerm | [DM-14](domain.md#dm-14) | admin, reviewer (promote); system (candidates) | [ADP-03](ports-adapters.md#adp-03), [ADP-02](ports-adapters.md#adp-02), [EVT-05](domain.md#evt-05) handler | none |
| [CMD-24](#cmd-24) | PublishRuleSet | [DM-15](domain.md#dm-15) | admin | [ADP-03](ports-adapters.md#adp-03) | none |
| [CMD-25](#cmd-25) | RequestReprocessing | [DM-17](domain.md#dm-17) (new) | admin; system for inheritance | [ADP-03](ports-adapters.md#adp-03), [ADP-01](ports-adapters.md#adp-01), [EVT-04](domain.md#evt-04) handler | [EVT-10](domain.md#evt-10) |
| [CMD-27](#cmd-27) | RecordRunLifecycle | [DM-17](domain.md#dm-17) | worker | [WF-01.3](ingestion.md#wf-01-3), [WF-03](ingestion.md#wf-03) | [EVT-10](domain.md#evt-10) |
| [CMD-28](#cmd-28) | SelectReprocessingPage | [DM-02](domain.md#dm-02) (batch) | worker | [WF-03.2](ingestion.md#wf-03-2) | none |
| [CMD-29](#cmd-29) | ClaimReviewTask | [DM-13](domain.md#dm-13) | reviewer | [ADP-02](ports-adapters.md#adp-02), [ADP-01](ports-adapters.md#adp-01) | none |

Commands with no event are still audited when a human or agent issues them ([CMD-26.10](#cmd-26-10)).

<a id="cmd-26"></a>
### CMD-26 Command handler contract

Rules that every command handler follows. Each command section below states only what differs or what is specific to it.

- <a id="cmd-26-1"></a>**CMD-26.1 Shape.** A command is a frozen Pydantic v2 model `<Name>Command` in `docstore.application.commands.<area>`, subclassing `Command[<Name>Result]` imported from `docstore.ports` ([MOD-04.1](overview.md#mod-04-1)). Its handler is a class with `__call__(cmd, ctx) -> <Name>Result`. `ctx` is an `ExecutionContext` (also from `docstore.ports`) holding the principal ([DM-16.1](domain.md#dm-16-1)), the acting agent, `run_id`, `workflow_id`, a correlation ID and a clock.
- <a id="cmd-26-2"></a>**CMD-26.2 Authorisation.** The handler checks the principal's roles against the Callers column before it loads anything, and fails with `Forbidden`. For document- or matter-scoped commands, a target the principal cannot see ([DM-16.7](domain.md#dm-16-7)) fails with `NotFound`, never `Forbidden`, so that walled items look nonexistent.
- <a id="cmd-26-3"></a>**CMD-26.3 One aggregate per transaction.** This applies [ARCH-06](overview.md#arch-06). A command modifies at most one existing aggregate. It may also create new aggregates (review tasks, child source items) in the same transaction. Run counters ([DM-17.2](domain.md#dm-17-2)) are updated with atomic SQL increments, not by loading the run. The named exceptions, each committed as one transaction:
  - [CMD-04](#cmd-04): a page of source items and the cursor commit atomically.
  - [CMD-21](#cmd-21), [CMD-22](#cmd-22): imports commit per file ([CMD-21.3](#cmd-21-3)).
  - [CMD-05](#cmd-05), [CMD-07](#cmd-07): the Document and its DuplicateSet ([DM-04](domain.md#dm-04)) change together, so a document is never a member of a set that does not list it.
  - [CMD-08](#cmd-08): the Thread ([DM-06](domain.md#dm-06)) and the document's `thread_id` change together, so thread membership and the back-reference cannot diverge.
  - [CMD-09](#cmd-09): the DuplicateSet and the signature stored on the Document change together, for the same reason as [CMD-05](#cmd-05).
  - [CMD-14](#cmd-14): one provider batch writes vectors for chunks of several documents, so that a crash loses at most that batch ([CMD-14.3](#cmd-14-3)); only vector columns are written.
  - [CMD-28](#cmd-28): a page of source-item stage resets commits atomically, as for [CMD-04](#cmd-04).
  - [CMD-16](#cmd-16): the SourceItem stage and the Document status move to quarantine together, so a quarantined stage never leaves a document looking healthy.
- <a id="cmd-26-4"></a>**CMD-26.4 Two phases for slow calls.** Calls to the connector, blob store, extractor, OCR, LLM or embedding API happen in a compute phase with no open transaction. The apply phase then opens the unit of work ([PORT-04](ports-adapters.md#port-04)), reloads the aggregate, re-checks preconditions such as `text_sha256` and stage status, and saves.
- <a id="cmd-26-5"></a>**CMD-26.5 Idempotency.** Each command states an idempotency key. Executing it again with the same key returns the earlier result and has no further side effects, which makes re-executing a workflow step safe ([ING-18](ingestion.md#ing-18)). This is implemented with natural-key upserts and version-stamp checks, not with a separate dedup table.
- <a id="cmd-26-6"></a>**CMD-26.6 Optimistic concurrency.** A save is `UPDATE ... WHERE version = :expected`. On conflict the handler retries the apply phase once, then raises `ConcurrencyConflict`, which is classified as transient.
- <a id="cmd-26-7"></a>**CMD-26.7 Results.** A command returns only IDs, status and counts ([ARCH-04](overview.md#arch-04)). Callers that need data issue a query.
- <a id="cmd-26-8"></a>**CMD-26.8 Errors.** Handlers raise `ValidationFailed`, `NotFound`, `Forbidden`, `Conflict`, `InvariantViolation` (from the domain), `ExternalTransient`, `ExternalRateLimited` and `ExternalPermanent`. These map to the classes of [XC-05](overview.md#xc-05). Inbound adapters translate them: HTTP 422/404/403/409, and an MCP tool error with the same code.
- <a id="cmd-26-9"></a>**CMD-26.9 Events.** Events recorded by aggregates are collected by the unit of work and dispatched after commit through [PORT-19](ports-adapters.md#port-19). A handler never dispatches before commit.
- <a id="cmd-26-10"></a>**CMD-26.10 Audit of issued commands.** A command issued by a `human` or `agent` principal is audited with its name, its input (text fields truncated to 500 characters) and its outcome, whether or not it emits an event. The writer and its transaction are defined in [XC-08.2](overview.md#xc-08-2) ([PORT-04.5](ports-adapters.md#port-04-5), [PORT-01.3](ports-adapters.md#port-01-3)). Commands from `worker` are logged only.

Refs: [ARCH-04](overview.md#arch-04), [ARCH-06](overview.md#arch-06), [PORT-01](ports-adapters.md#port-01), [XC-05](overview.md#xc-05)

## Commands: sources and runs

<a id="cmd-01"></a>
### CMD-01 RegisterSource

- <a id="cmd-01-1"></a>**CMD-01.1 Input.** `source_id` (absent for create), `name`, `kind`, `settings`, `unassigned_access_group`, `consent_reference` ([DM-01.5](domain.md#dm-01-5)), `enabled`. The settings schema depends on `kind`. `filesystem`: `root_path`, `include_globs[]`, `exclude_globs[]`, `follow_symlinks=false`. `sharepoint`: `tenant_id`, `client_id`, `credential_ref` (env var name, never the secret), `site_urls[]`, `libraries[]`, `export_path` (folder-export fallback). `pst`: `paths[]` (files or globs), `custodian_person_id`.
- <a id="cmd-01-2"></a>**CMD-01.2 Output.** `source_id`, and `cursor_reset: bool`.
- <a id="cmd-01-3"></a>**CMD-01.3 Behaviour.** The handler validates the settings, then calls `check()` on the connector ([PORT-16](ports-adapters.md#port-16)) outside the transaction: the root path is readable, the Graph token resolves and a site is reachable, or the PST files open. It then upserts the source and applies the cursor rule [DM-01.3](domain.md#dm-01-3).
- <a id="cmd-01-4"></a>**CMD-01.4 Idempotency.** Key: `name`. An unchanged payload is a no-op.

Refs: [DM-01](domain.md#dm-01), [PORT-07](ports-adapters.md#port-07), [PORT-16](ports-adapters.md#port-16), [PORT-04](ports-adapters.md#port-04)

<a id="cmd-02"></a>
### CMD-02 StartIngestionRun

- <a id="cmd-02-1"></a>**CMD-02.1 Input.** `source_id`, `scope` (optional narrowing: path prefixes, site or library IDs, PST files, `modified_since`), `mode` (`incremental` by default, or `full`), `pipeline_version` (current by default), `run_key` (optional).
- <a id="cmd-02-2"></a>**CMD-02.2 Output.** `run_id`, `workflow_id`, `created: bool`.
- <a id="cmd-02-3"></a>**CMD-02.3 Behaviour.** The handler checks [DM-01.4](domain.md#dm-01-4) and [DM-17.6](domain.md#dm-17-6). If another run on the source is active, it raises `Conflict` carrying that run's ID. Otherwise it creates the run as `pending` and commits. It then starts [WF-01](ingestion.md#wf-01) through [PORT-18](ports-adapters.md#port-18) with `workflow_id = "run:" + run_id`. The run becomes `running` when the workflow starts.
- <a id="cmd-02-4"></a>**CMD-02.4 Idempotency.** Key: `run_key`. When the caller omits it, `run_key` is the sha256 of (`source_id`, `scope`, `mode`, `pipeline_version`, a caller-generated UUID). The CLI prints the key so a retry can pass it again. A repeated key returns the existing run, whatever its state. Starting the workflow again is safe because workflow IDs are deterministic.

Refs: [DM-17](domain.md#dm-17), [DM-01](domain.md#dm-01), [PORT-07](ports-adapters.md#port-07), [PORT-18](ports-adapters.md#port-18), [EVT-10](domain.md#evt-10), [WF-01](ingestion.md#wf-01)

<a id="cmd-03"></a>
### CMD-03 ControlIngestionRun

- <a id="cmd-03-1"></a>**CMD-03.1 Input.** `run_id`, `action` ∈ {`pause`, `resume`, `cancel`}, `reason`.
- <a id="cmd-03-2"></a>**CMD-03.2 Output.** `run_id`, `state`.
- <a id="cmd-03-3"></a>**CMD-03.3 Behaviour.** The handler validates the transition against [DM-17.3](domain.md#dm-17-3), saves the new state, and after commit calls [PORT-18](ports-adapters.md#port-18) to cancel or resume the run's workflows. What pause means for in-flight steps is defined in [WF-05](ingestion.md#wf-05).
- <a id="cmd-03-4"></a>**CMD-03.4 Idempotency.** An action whose target state equals the current state returns success and does nothing.

Refs: [DM-17](domain.md#dm-17), [PORT-07](ports-adapters.md#port-07), [PORT-18](ports-adapters.md#port-18), [EVT-10](domain.md#evt-10), [WF-05](ingestion.md#wf-05)

<a id="cmd-27"></a>
### CMD-27 RecordRunLifecycle

- <a id="cmd-27-1"></a>**CMD-27.1 Input.** `run_id`, `target_state` ∈ {`running`, `completed`, `failed`}, `reason`.
- <a id="cmd-27-2"></a>**CMD-27.2 Output.** `run_id`, `state`.
- <a id="cmd-27-3"></a>**CMD-27.3 Behaviour.** System-only; issued by the first and last steps of a run workflow ([WF-01.3](ingestion.md#wf-01-3)) as `svc-ingest`. The handler validates the transition against [DM-17.3](domain.md#dm-17-3) and, for `completed`, the guard of [DM-17.5](domain.md#dm-17-5); a run that is `cancelled` or `paused` is left unchanged and the current state is returned.
- <a id="cmd-27-4"></a>**CMD-27.4 Idempotency.** Key: (`run_id`, `target_state`). A run already in the target state returns success and emits nothing.

Refs: [DM-17](domain.md#dm-17), [PORT-07](ports-adapters.md#port-07), [EVT-10](domain.md#evt-10), [WF-01](ingestion.md#wf-01)

<a id="cmd-04"></a>
### CMD-04 DiscoverSourceItems

- <a id="cmd-04-1"></a>**CMD-04.1 Input.** `run_id`, `source_id`, `cursor` (opaque; null means start from the source's saved cursor or from scratch in `full` mode), `page_size` (default 500).
- <a id="cmd-04-2"></a>**CMD-04.2 Output.** `next_cursor`, `exhausted`, counts `new` / `changed` / `unchanged` / `removed`, and `to_process[]` (source item IDs that are new or changed).
- <a id="cmd-04-3"></a>**CMD-04.3 Behaviour.** The handler calls `list_page(cursor, page_size)` on the connector outside the transaction. In one transaction it then upserts each listed item by (`source_id`, `source_key`) and compares fingerprints ([DM-02.2](domain.md#dm-02-2)). New items are inserted with all stages `pending`. Changed items get a new revision ([DM-02.8](domain.md#dm-02-8)). Unchanged items are only touched (`last_run_id`). Items reported with `excluded_by` get stage `fetch` = `excluded` ([DM-02.6](domain.md#dm-02-6)) and are not returned in `to_process`. Items the connector reports as deleted get `removed_at_source_at`, and the matching document location is marked removed. The source cursor is saved in the same transaction ([DM-01.2](domain.md#dm-01-2)), and the run's `discovered_total` is incremented.
- <a id="cmd-04-4"></a>**CMD-04.4 Idempotency.** Key: (`run_id`, input cursor). Upserts make a replayed page produce the same rows and the same `next_cursor`.

Refs: [DM-02](domain.md#dm-02), [DM-01](domain.md#dm-01), [PORT-16](ports-adapters.md#port-16), [PORT-07](ports-adapters.md#port-07), [ING-01](ingestion.md#ing-01)

## Commands: document pipeline

<a id="cmd-05"></a>
### CMD-05 IngestSourceItem

- <a id="cmd-05-1"></a>**CMD-05.1 Input.** `source_item_id`, `run_id`.
- <a id="cmd-05-2"></a>**CMD-05.2 Output.** `document_id`, `outcome` ∈ {`registered`, `location_added`, `email_identity_duplicate`, `unchanged`}.
- <a id="cmd-05-3"></a>**CMD-05.3 Behaviour.** In the compute phase the handler streams the item's bytes from the connector, computes sha256 while streaming, and calls `put` on [PORT-17](ports-adapters.md#port-17) unless `exists` returns true. Items larger than the configured `max_item_bytes` (default 2 GiB) raise `ExternalPermanent`. In the apply phase:
  1. If a document with this sha256 exists, the item is added as a location ([DM-03.11](domain.md#dm-03-11)).
  2. Otherwise, for e-mail items whose connector supplied headers (PST), the e-mail identity hash ([DM-04.2](domain.md#dm-04-2)) is computed. If it matches, the item is registered as a `duplicate` member of that set. For EML and MSG files this check happens later, in [CMD-07](#cmd-07).
  3. Otherwise a new document is registered with status `registered`. Its family fields come from the source item's parent ([DM-03.5](domain.md#dm-03-5)).

  The `fetch` and `dedup` stages are set to `done` with `document_id`.
- <a id="cmd-05-4"></a>**CMD-05.4 Idempotency.** Key: (`source_item_id`, `revision`). A source item whose `fetch` stage is already `done` returns `unchanged` with its `document_id`.

Refs: [DM-03](domain.md#dm-03), [DM-04](domain.md#dm-04), [DM-02](domain.md#dm-02), [PORT-16](ports-adapters.md#port-16), [PORT-17](ports-adapters.md#port-17), [PORT-05](ports-adapters.md#port-05), [EVT-01](domain.md#evt-01), [EVT-03](domain.md#evt-03), [ADR-11](decisions.md#adr-11)

<a id="cmd-06"></a>
### CMD-06 ExpandContainer

- <a id="cmd-06-1"></a>**CMD-06.1 Input.** `document_id`, `source_item_id`.
- <a id="cmd-06-2"></a>**CMD-06.2 Output.** `child_source_item_ids[]`, `skipped_children` (count above limits).
- <a id="cmd-06-3"></a>**CMD-06.3 Behaviour.** The handler applies to e-mails (EML, MSG and PST messages), ZIP archives and Office or PDF files with embedded objects; for other documents the stage is `skipped`. It uses `Extraction.children` from `extract()` ([PORT-14.3](ports-adapters.md#port-14-3)) and stores each child's bytes through [PORT-17](ports-adapters.md#port-17). It then creates child source items with keys `<parent key>/att/<ordinal>` and `parent_source_item_id`; each child then runs its own pipeline. The limits are depth 5 ([DM-03.12](domain.md#dm-03-12)), 1,000 children per parent and an expansion ratio of 100× (zip-bomb guard). Exceeding a limit quarantines the parent's `expand` stage, but children already produced are kept.
- <a id="cmd-06-4"></a>**CMD-06.4 Idempotency.** Key: (`document_id`, `parser_version`). Child keys are deterministic, so a re-run upserts the same children.

Refs: [DM-03](domain.md#dm-03), [DM-02](domain.md#dm-02), [PORT-14](ports-adapters.md#port-14), [PORT-17](ports-adapters.md#port-17), [PORT-07](ports-adapters.md#port-07), [ING-04](ingestion.md#ing-04)

<a id="cmd-07"></a>
### CMD-07 ExtractDocumentText

- <a id="cmd-07-1"></a>**CMD-07.1 Input.** `document_id`, `force_ocr` (default false, for reprocessing).
- <a id="cmd-07-2"></a>**CMD-07.2 Output.** `text_sha256`, `char_count`, `page_count`, `ocr_applied`, `outcome` ∈ {`extracted`, `no_text`, `email_identity_duplicate`, `unchanged`}.
- <a id="cmd-07-3"></a>**CMD-07.3 Behaviour.** The handler loads the original from [PORT-17](ports-adapters.md#port-17) and calls `extract()` on [PORT-14](ports-adapters.md#port-14), which returns text, native metadata, e-mail headers and `has_text_layer`. **OCR rule:** for PDFs and images, OCR runs through [PORT-15](ports-adapters.md#port-15) when `has_text_layer` is false, when the extracted text has fewer than 50 non-whitespace characters per page on average, or when `force_ocr` is set. Pages that have text are kept, and only the others are OCRed. The normalised UTF-8 text (NFC, form feeds marking page breaks) is stored as a blob. The apply phase sets the [DM-03.3](domain.md#dm-03-3), [DM-03.4](domain.md#dm-03-4) and [DM-03.7](domain.md#dm-03-7) fields and stamps `parser_version` and `ocr_version`. It then checks e-mail identity ([DM-04.2](domain.md#dm-04-2)) for EML and MSG and, on a match, applies [DM-04.6](domain.md#dm-04-6). Finally it moves the document to `text_ready`. Empty text sets `no_text`, and the document continues on metadata only.
- <a id="cmd-07-4"></a>**CMD-07.4 Idempotency.** Key: (`document_id`, `parser_version`, `ocr_version`). If the stamps already match and `force_ocr` is false, the result is `unchanged`. Re-extraction that yields the same `text_sha256` does not invalidate later stages.

Refs: [DM-03](domain.md#dm-03), [PORT-14](ports-adapters.md#port-14), [PORT-15](ports-adapters.md#port-15), [PORT-17](ports-adapters.md#port-17), [PORT-05](ports-adapters.md#port-05), [EVT-02](domain.md#evt-02), [ADR-07](decisions.md#adr-07)

<a id="cmd-08"></a>
### CMD-08 LinkFamilyAndThread

- <a id="cmd-08-1"></a>**CMD-08.1 Input.** `document_id`.
- <a id="cmd-08-2"></a>**CMD-08.2 Output.** `family_id`, `thread_id` (null for non-e-mail), `merged_thread_ids[]`.
- <a id="cmd-08-3"></a>**CMD-08.3 Behaviour.** For every document, the handler confirms `family_id` from the parent chain ([DM-03.12](domain.md#dm-03-12)). For `email` documents it applies the linking order [DM-06.2](domain.md#dm-06-2), looking up candidate threads by `message_id`, conversation index and normalised subject through `find_thread` of [PORT-05](ports-adapters.md#port-05). It then attaches the message, creating or merging threads ([DM-06.3](domain.md#dm-06-3)), and stores `thread_id` on the document. Attachments are not thread members; they reach the thread through their family.
- <a id="cmd-08-4"></a>**CMD-08.4 Idempotency.** Key: `document_id`. Membership has set semantics, so attaching again is a no-op.

Refs: [DM-06](domain.md#dm-06), [DM-03](domain.md#dm-03), [PORT-05](ports-adapters.md#port-05)

<a id="cmd-09"></a>
### CMD-09 DetectNearDuplicates

- <a id="cmd-09-1"></a>**CMD-09.1 Input.** `document_id`.
- <a id="cmd-09-2"></a>**CMD-09.2 Output.** `set_id` (or null), `canonical_document_id`, `similarity`.
- <a id="cmd-09-3"></a>**CMD-09.3 Behaviour.** The handler computes the MinHash signature ([DM-04.3](domain.md#dm-04-3)) and stores it on the document, skipping documents under 50 words. It finds candidates that share an LSH band through `near_duplicate_candidates` of [PORT-05](ports-adapters.md#port-05) ([PORT-05.3](ports-adapters.md#port-05-3)) and verifies the estimated Jaccard against the threshold. It then joins the best-matching set or creates one, and re-selects the canonical member by [DM-04.4](domain.md#dm-04-4). The lookup runs on the `docstore_app` connection, which is not under RLS ([XC-01.6](overview.md#xc-01-6)), and returns only IDs; the result is never returned to a principal.
- <a id="cmd-09-4"></a>**CMD-09.4 Idempotency.** Key: (`document_id`, `text_sha256`, `near_dup_algo_version`).

Refs: [DM-04](domain.md#dm-04), [PORT-05](ports-adapters.md#port-05), [EVT-03](domain.md#evt-03)

<a id="cmd-10"></a>
### CMD-10 ApplyAssignmentRules

- <a id="cmd-10-1"></a>**CMD-10.1 Input.** `document_id`, `ruleset_version` (the published version by default).
- <a id="cmd-10-2"></a>**CMD-10.2 Output.** `proposals` (count by target kind), `ruleset_version`.
- <a id="cmd-10-3"></a>**CMD-10.3 Behaviour.** The handler builds `DocumentFacts`: paths and SharePoint site, library and columns from every location, including those of `duplicate` members; sender and recipient domains; `mailbox_owner`; PST folder path; file name; subject; and the first 2,000 characters of text. It loads the published rule set through [PORT-09](ports-adapters.md#port-09) and reference data through [PORT-06](ports-adapters.md#port-06), then evaluates the rules ([DM-15.4](domain.md#dm-15-4)). It adds `inherited` proposals from three places: the family parent's accepted matter and client, the thread matter ([DM-06.4](domain.md#dm-06-4), evaluated on the thread's current accepted assignments, not read from the stored value), and, for an accepted matter, its client, case type and responsible lawyer. The proposals are written as `proposed`, replacing earlier `rule`/`inherited` proposals that were not accepted. Finally it stamps `ruleset_version`. It does not accept anything; that is [CMD-12](#cmd-12).
- <a id="cmd-10-4"></a>**CMD-10.4 Idempotency.** Key: (`document_id`, `ruleset_version`, sha256 of `DocumentFacts`).

Refs: [DM-03](domain.md#dm-03), [DM-10](domain.md#dm-10), [DM-15](domain.md#dm-15), [PORT-05](ports-adapters.md#port-05), [PORT-06](ports-adapters.md#port-06), [PORT-09](ports-adapters.md#port-09), [EVT-04](domain.md#evt-04), [ADR-08](decisions.md#adr-08)

<a id="cmd-11"></a>
### CMD-11 EnrichDocument

- <a id="cmd-11-1"></a>**CMD-11.1 Input.** `document_id`.
- <a id="cmd-11-2"></a>**CMD-11.2 Output.** `status` ([DM-12.1](domain.md#dm-12-1)), `fields_written[]`, `new_tag_candidates[]`.
- <a id="cmd-11-3"></a>**CMD-11.3 Behaviour.** The stage is `skipped` for `duplicate` and `no_text` documents. In the compute phase the handler builds the prompt input of [LLM-02](ingestion.md#llm-02): an excerpt, metadata, candidate matters from this document's [CMD-10](#cmd-10) proposals, and the allowed taxonomy values from [PORT-09](ports-adapters.md#port-09). It calls [PORT-13](ports-adapters.md#port-13), then validates and coerces the output under [LLM-05](ingestion.md#llm-05). In the apply phase it writes a new Enrichment that respects locked fields ([DM-12.4](domain.md#dm-12-4)), with provenance per [DM-11.3](domain.md#dm-11-3). Matter candidates, case type and staff people from the output become `llm` assignment proposals. Unknown tags are listed as candidates for the [EVT-05](domain.md#evt-05) handler. An output that is still invalid after repair is stored with `status = invalid_output`, and [CMD-12](#cmd-12) then opens an `invalid_llm_output` task.
- <a id="cmd-11-4"></a>**CMD-11.4 Idempotency.** Key: (`document_id`, `text_sha256`, `prompt_version`, `llm_model`, `taxonomy_version`). If an enrichment with this key exists, the result is returned without calling the LLM.

Refs: [DM-12](domain.md#dm-12), [DM-03](domain.md#dm-03), [PORT-13](ports-adapters.md#port-13), [PORT-05](ports-adapters.md#port-05), [PORT-09](ports-adapters.md#port-09), [EVT-05](domain.md#evt-05), [LLM-02](ingestion.md#llm-02), [LLM-03](ingestion.md#llm-03), [LLM-05](ingestion.md#llm-05)

<a id="cmd-12"></a>
### CMD-12 ReconcileAssignments

- <a id="cmd-12-1"></a>**CMD-12.1 Input.** `document_id`.
- <a id="cmd-12-2"></a>**CMD-12.2 Output.** `accepted[]` and `rejected[]` (target kind and target), `review_task_ids[]`, `access_keys_changed: bool`.
- <a id="cmd-12-3"></a>**CMD-12.3 Behaviour.** The handler loads the document and the open review tasks for it. It applies the reconciliation policy [DM-10.6](domain.md#dm-10-6) to the assignments, using the bands of [TAX-04.3](domain.md#tax-04-3), and applies the same bands to the `doc_type` and `tags` enrichment values ([DM-12.2](domain.md#dm-12-2)). When accepted matters change, it recomputes access keys on the document and its chunks ([DM-03.14](domain.md#dm-03-14)). It creates review tasks as new aggregates, respecting [DM-13.4](domain.md#dm-13-4). If the document is already `indexed`, the chunk filter keys are refreshed in the same transaction through [PORT-10](ports-adapters.md#port-10), so search never sees stale access keys.
- <a id="cmd-12-4"></a>**CMD-12.4 Idempotency.** Key: (`document_id`, `version`). The policy is deterministic over the same proposals, and the open-task uniqueness rule prevents duplicate tasks.

Refs: [DM-10](domain.md#dm-10), [DM-13](domain.md#dm-13), [PORT-05](ports-adapters.md#port-05), [PORT-08](ports-adapters.md#port-08), [PORT-10](ports-adapters.md#port-10), [EVT-04](domain.md#evt-04), [EVT-07](domain.md#evt-07), [TAX-04](domain.md#tax-04), [ADR-08](decisions.md#adr-08)

<a id="cmd-13"></a>
### CMD-13 ChunkDocument

- <a id="cmd-13-1"></a>**CMD-13.1 Input.** `document_id`.
- <a id="cmd-13-2"></a>**CMD-13.2 Output.** `generation`, `chunk_count`, `outcome` ∈ {`chunked`, `unchanged`, `skipped`}.
- <a id="cmd-13-3"></a>**CMD-13.3 Behaviour.** The stage is `skipped` for `duplicate` and `no_text` documents. The handler loads the text and splits it with the parameters of [ADR-12](decisions.md#adr-12), cutting at page breaks, headings and paragraphs before falling back to sentence and token limits. In e-mails, quoted earlier messages are removed when the quoted message is itself in the same thread. Section hints are set. The chunks are written as a new pending generation ([DM-05.5](domain.md#dm-05-5)), copying the filter keys from the document ([DM-05.3](domain.md#dm-05-3)), and `chunker_version` is stamped.
- <a id="cmd-13-4"></a>**CMD-13.4 Idempotency.** Key: (`document_id`, `text_sha256`, `chunker_version`). If the active or pending generation already has this key, the result is `unchanged`.

Refs: [DM-05](domain.md#dm-05), [PORT-05](ports-adapters.md#port-05), [ADR-12](decisions.md#adr-12)

<a id="cmd-14"></a>
### CMD-14 EmbedChunks

- <a id="cmd-14-1"></a>**CMD-14.1 Input.** `document_ids[]` (one or several, so the workflow can fill provider batches, [WF-04](ingestion.md#wf-04)), `embedding_model` (the configured model by default).
- <a id="cmd-14-2"></a>**CMD-14.2 Output.** `embedded` (chunk count), `tokens`, `requests`.
- <a id="cmd-14-3"></a>**CMD-14.3 Behaviour.** The handler selects chunks of the pending generation, or of the active generation during a model change, whose `embedding` is null or whose `embedding_model` differs from the target. It packs them into requests within the provider limits reported by [PORT-12](ports-adapters.md#port-12) and calls `embed_documents(texts, token_counts)` outside the transaction. It checks that the dimension matches ([DM-05.6](domain.md#dm-05-6)), then writes vectors, `embedding_model` and `embedded_at` per request batch. Each batch is committed on its own, so a crash loses at most the batch in flight ([SC-08](../success-criteria.md#sc-08)). HTTP 429 raises `ExternalRateLimited` with the provider's retry-after value.
- <a id="cmd-14-4"></a>**CMD-14.4 Idempotency.** Key: the set of (`chunk_id`, `embedding_model`). Chunks that are already embedded are never sent again.

Refs: [DM-05](domain.md#dm-05), [PORT-12](ports-adapters.md#port-12), [PORT-05](ports-adapters.md#port-05), [ADR-06](decisions.md#adr-06)

<a id="cmd-15"></a>
### CMD-15 IndexDocument

- <a id="cmd-15-1"></a>**CMD-15.1 Input.** `document_id`.
- <a id="cmd-15-2"></a>**CMD-15.2 Output.** `status`, `generation`, `chunk_count`.
- <a id="cmd-15-3"></a>**CMD-15.3 Behaviour.** The handler checks the searchable precondition [DM-03.13](domain.md#dm-03-13) on the pending generation; if no pending generation exists, it only refreshes filter keys. In one transaction it promotes the pending generation to active, deletes the old generation's chunks, writes the BM25 and vector entries and filter keys through [PORT-10](ports-adapters.md#port-10), sets `status = indexed`, and stamps `embedding_model`. A `duplicate` document is never indexed.
- <a id="cmd-15-4"></a>**CMD-15.4 Idempotency.** Key: (`document_id`, `generation`). Indexing an active generation again is a no-op.

Refs: [DM-03](domain.md#dm-03), [DM-05](domain.md#dm-05), [PORT-10](ports-adapters.md#port-10), [EVT-09](domain.md#evt-09), [ADR-03](decisions.md#adr-03)

<a id="cmd-16"></a>
### CMD-16 RecordStepFailure

- <a id="cmd-16-1"></a>**CMD-16.1 Input.** `source_item_id`, `run_id`, `stage` ([DM-02.5](domain.md#dm-02-5)), `error_class` ([XC-05](overview.md#xc-05)), `message` (truncated to 2,000 characters), `exception_type`, `attempt`.
- <a id="cmd-16-2"></a>**CMD-16.2 Output.** `stage_status` ∈ {`failed`, `quarantined`}, `retry_allowed: bool`.
- <a id="cmd-16-3"></a>**CMD-16.3 Behaviour.** The handler records the attempt on the source item's stage. It quarantines the stage ([DM-02.9](domain.md#dm-02-9)) when the class is permanent or poison, or when the attempts reach the class maximum of [XC-05](overview.md#xc-05). If the stage belongs to a registered document, the document also becomes `quarantined`. The run's `failed` or `quarantined` counter is incremented atomically ([CMD-26.3](#cmd-26-3)).
- <a id="cmd-16-4"></a>**CMD-16.4 Idempotency.** Key: (`source_item_id`, `stage`, `attempt`).

Refs: [DM-02](domain.md#dm-02), [DM-17](domain.md#dm-17), [PORT-07](ports-adapters.md#port-07), [XC-05](overview.md#xc-05)

## Commands: review and override

<a id="cmd-17"></a>
### CMD-17 OverrideAssignment

- <a id="cmd-17-1"></a>**CMD-17.1 Input.** `document_id`, `target_kind`, `role`, `action` ∈ {`set`, `reject`}, `target_id` (for `set`), `reason` (required, at most 1,000 characters).
- <a id="cmd-17-2"></a>**CMD-17.2 Output.** `document_id`, `access_keys_changed: bool`.
- <a id="cmd-17-3"></a>**CMD-17.3 Behaviour.** The principal must see the document and, for `set` on a matter or client, the target too; otherwise the result is `NotFound` ([CMD-26.2](#cmd-26-2)). The handler records a `human` assignment as `accepted` (or a `human` rejection of the currently accepted target) with provenance per [DM-11.3](domain.md#dm-11-3). It supersedes the automated assignments it replaces ([DM-10.2](domain.md#dm-10-2)), recomputes access keys and refreshes the index keys as in [CMD-12.3](#cmd-12-3). Family children and thread members are updated through the [EVT-04](domain.md#evt-04) handler.
- <a id="cmd-17-4"></a>**CMD-17.4 Idempotency.** Setting the same target that is already accepted with method `human` is a no-op.

Refs: [DM-10](domain.md#dm-10), [DM-03](domain.md#dm-03), [PORT-05](ports-adapters.md#port-05), [PORT-06](ports-adapters.md#port-06), [EVT-04](domain.md#evt-04), [EVT-06](domain.md#evt-06)

<a id="cmd-18"></a>
### CMD-18 OverrideEnrichment

- <a id="cmd-18-1"></a>**CMD-18.1 Input.** `document_id`, `field` (an [LLM-03](ingestion.md#llm-03) field name), `value`, `reason`.
- <a id="cmd-18-2"></a>**CMD-18.2 Output.** `document_id`, `field`.
- <a id="cmd-18-3"></a>**CMD-18.3 Behaviour.** The handler validates the value against [DM-12.3](domain.md#dm-12-3); taxonomy terms must be active, which it checks through [PORT-09](ports-adapters.md#port-09). It sets the field with `human` provenance and `locked = true` ([DM-12.4](domain.md#dm-12-4)). If the field is `doc_type` or `tags`, it refreshes the chunk filter keys.
- <a id="cmd-18-4"></a>**CMD-18.4 Idempotency.** An identical value that is already locked is a no-op.

Refs: [DM-12](domain.md#dm-12), [PORT-05](ports-adapters.md#port-05), [PORT-09](ports-adapters.md#port-09), [EVT-06](domain.md#evt-06)

<a id="cmd-19"></a>
### CMD-19 ResolveReviewTask

- <a id="cmd-19-1"></a>**CMD-19.1 Input.** `task_id`, `decision` ∈ {`accept_candidate`, `override`, `dismiss`, `superseded`}, `candidate_index` or `value`, `comment`.
- <a id="cmd-19-2"></a>**CMD-19.2 Output.** `task_id`, `state`.
- <a id="cmd-19-3"></a>**CMD-19.3 Behaviour.** For `accept_candidate` and `override`, the handler first runs [CMD-17](#cmd-17) or [CMD-18](#cmd-18) as the resolver, in its own transaction. It then moves the task to `resolved` in a second transaction ([DM-13.5](domain.md#dm-13-5)). If the process crashes between the two, a retry finds the override already applied (a no-op) and only resolves the task. `superseded` is issued only by the [EVT-06](domain.md#evt-06) handler, and `dismiss` requires a comment.
- <a id="cmd-19-4"></a>**CMD-19.4 Idempotency.** Resolving a task that is already terminal with the same decision returns success; a different decision raises `Conflict`.

Refs: [DM-13](domain.md#dm-13), [PORT-08](ports-adapters.md#port-08), [CMD-17](#cmd-17), [CMD-18](#cmd-18), [EVT-08](domain.md#evt-08)

<a id="cmd-20"></a>
### CMD-20 FlagForReview

- <a id="cmd-20-1"></a>**CMD-20.1 Input.** `document_id`, `subject` ([DM-13.1](domain.md#dm-13-1)), `reason_text` (required, at most 2,000 characters), `suggested_value` (optional).
- <a id="cmd-20-2"></a>**CMD-20.2 Output.** `task_id`, `created: bool`.
- <a id="cmd-20-3"></a>**CMD-20.3 Behaviour.** The document must be visible to the caller. The handler opens a task with reason `agent_flag` or `human_flag`, depending on the principal kind. A suggested value becomes a candidate with method `human` for people, or `agent` for agents ([DM-11.3](domain.md#dm-11-3)). If a task on the subject is already open, it appends the comment instead ([DM-13.4](domain.md#dm-13-4)). Agents are limited to 30 flags per minute per principal, and above that the handler returns `ExternalRateLimited`.
- <a id="cmd-20-4"></a>**CMD-20.4 Idempotency.** Key: (`document_id`, `subject`, caller, sha256 of `reason_text`).

Refs: [DM-13](domain.md#dm-13), [PORT-08](ports-adapters.md#port-08), [EVT-07](domain.md#evt-07)

<a id="cmd-29"></a>
### CMD-29 ClaimReviewTask

- <a id="cmd-29-1"></a>**CMD-29.1 Input.** `task_id`, `action` ∈ {`claim`, `release`}.
- <a id="cmd-29-2"></a>**CMD-29.2 Output.** `task_id`, `state`, `assignee_id`, `claimed_until`.
- <a id="cmd-29-3"></a>**CMD-29.3 Behaviour.** `claim` moves an `open` task, or a `claimed` task whose claim has expired, to `claimed` with the caller as `assignee_id` and `claimed_until` per [DM-13.3](domain.md#dm-13-3). A task claimed by someone else with an unexpired claim raises `Conflict`. `release` returns the caller's own claim to `open`. Expiry needs no command: a claim past `claimed_until` counts as `open` for [QRY-09](#qry-09) and for the next `claim`.
- <a id="cmd-29-4"></a>**CMD-29.4 Idempotency.** Key: (`task_id`, `action`, principal). Claiming a task the caller already holds renews `claimed_until`; releasing a task that is not claimed by the caller returns success and does nothing.

Refs: [DM-13](domain.md#dm-13), [PORT-08](ports-adapters.md#port-08), [QRY-09](#qry-09)

## Commands: reference data and configuration

<a id="cmd-21"></a>
### CMD-21 ImportReferenceData

- <a id="cmd-21-1"></a>**CMD-21.1 Input.** `clients_csv`, `persons_csv`, `matters_csv` (any subset), a `column_map` per file, `missing_policy` ∈ {`keep`, `archive`} for matters absent from the file, `dry_run`.
- <a id="cmd-21-2"></a>**CMD-21.2 Output.** An import report: rows inserted, updated, unchanged and rejected per file, with row numbers and reasons, plus `matters_with_changed_assignment_inputs[]` (matters whose client, case type or responsible lawyer changed).
- <a id="cmd-21-3"></a>**CMD-21.3 Behaviour.** The handler parses the files (UTF-8, `;` or `,` detected) and validates each row against [DM-07](domain.md#dm-07), [DM-08](domain.md#dm-08) and [DM-09](domain.md#dm-09) invariants. It upserts by PMS ID in the order clients, persons, matters, all or nothing per file: any rejected row aborts the file (no rows written), and the result reports every rejected row with its reason. The changed-matter list is the input for an assign-stage [CMD-25](#cmd-25).
- <a id="cmd-21-4"></a>**CMD-21.4 Idempotency.** Upserts by PMS ID, so the same files produce only `unchanged` rows.

Refs: [DM-07](domain.md#dm-07), [DM-08](domain.md#dm-08), [DM-09](domain.md#dm-09), [PORT-06](ports-adapters.md#port-06)

<a id="cmd-22"></a>
### CMD-22 ImportAccessPolicies

- <a id="cmd-22-1"></a>**CMD-22.1 Input.** A JSON or CSV snapshot of principals (with `token_ref` for PoC tokens, [ADP-07](ports-adapters.md#adp-07)), groups and memberships, matter grants and walls ([DM-16](domain.md#dm-16)), plus `dry_run`.
- <a id="cmd-22-2"></a>**CMD-22.2 Output.** A diff report: principals, memberships, grants and walls added, changed and ended.
- <a id="cmd-22-3"></a>**CMD-22.3 Behaviour.** The snapshot replaces the current state. Entries missing from it get `valid_to = now`; they are never deleted, so the audit trail stays meaningful. Changes take effect for the next query because visibility is evaluated at query time ([XC-01](overview.md#xc-01)), so no reindex is needed.
- <a id="cmd-22-4"></a>**CMD-22.4 Idempotency.** Re-importing the same snapshot yields an empty diff.

Refs: [DM-16](domain.md#dm-16), [PORT-06](ports-adapters.md#port-06), [XC-01](overview.md#xc-01)

<a id="cmd-23"></a>
### CMD-23 UpsertTaxonomyTerm

- <a id="cmd-23-1"></a>**CMD-23.1 Input.** `vocabulary`, `code`, `action` ∈ {`create`, `update`, `deprecate`, `promote`, `propose`}, `labels`, `parent_code`, `synonyms[]`, `replaced_by`, `pms_code`.
- <a id="cmd-23-2"></a>**CMD-23.2 Output.** `code`, `status`, `taxonomy_version`.
- <a id="cmd-23-3"></a>**CMD-23.3 Behaviour.** The handler enforces [DM-14.2](domain.md#dm-14-2) to [DM-14.4](domain.md#dm-14-4). `propose` is used only by the [EVT-05](domain.md#evt-05) handler: it creates or counts a `candidate` and opens a promotion task at the threshold of [TAX-03.3](domain.md#tax-03-3). `promote` turns a candidate into an active term, and later enrichments match it; existing enrichments already hold the code. Every change bumps `taxonomy_version` ([DM-14.5](domain.md#dm-14-5)).
- <a id="cmd-23-4"></a>**CMD-23.4 Idempotency.** Key: (`vocabulary`, `code`, `action`, payload hash). `propose` is the exception: it counts once per (`code`, `document_id`).

Refs: [DM-14](domain.md#dm-14), [PORT-09](ports-adapters.md#port-09), [TAX-01](domain.md#tax-01), [TAX-02](domain.md#tax-02), [TAX-03](domain.md#tax-03)

<a id="cmd-24"></a>
### CMD-24 PublishRuleSet

- <a id="cmd-24-1"></a>**CMD-24.1 Input.** `rules_yaml` (a list of rules with the fields of [DM-15.2](domain.md#dm-15-2)), `comment`, `dry_run`, `sample_size` (default 1,000).
- <a id="cmd-24-2"></a>**CMD-24.2 Output.** `ruleset_version` (null on a dry run) and an impact preview: on a random sample of indexed documents, how many proposals would be added, removed or changed per target kind compared with the current version.
- <a id="cmd-24-3"></a>**CMD-24.3 Behaviour.** The handler validates against [DM-15.5](domain.md#dm-15-5), runs the preview, and when not a dry run publishes version n+1 and retires version n. Documents become stale for the `rules` stage by their `ruleset_version` stamp ([XC-06](overview.md#xc-06)); this command does not reprocess them.
- <a id="cmd-24-4"></a>**CMD-24.4 Idempotency.** Publishing rules identical to the current published version returns that version and creates no new one.

Refs: [DM-15](domain.md#dm-15), [PORT-09](ports-adapters.md#port-09), [XC-06](overview.md#xc-06)

<a id="cmd-25"></a>
### CMD-25 RequestReprocessing

- <a id="cmd-25-1"></a>**CMD-25.1 Input.** A `selection`: any of `source_id`, `matter_ids[]`, `document_ids[]`, `stale_for[]` (version stamps whose value differs from the current one, [DM-03.15](domain.md#dm-03-15)), or `quarantined_in_run`. `stages[]` from [DM-02.5](domain.md#dm-02-5) (later stages follow automatically). Plus `reason`, `dry_run` and `run_key`.
- <a id="cmd-25-2"></a>**CMD-25.2 Output.** `run_id` (null on a dry run), `selected_documents`, `estimated_llm_calls`, `estimated_embedding_tokens`.
- <a id="cmd-25-3"></a>**CMD-25.3 Behaviour.** The handler counts the selection and estimates cost through `count_selection` of [PORT-07](ports-adapters.md#port-07) ([PORT-07.3](ports-adapters.md#port-07-3)). It creates a `reprocess` run ([DM-17.1](domain.md#dm-17-1)) and starts [WF-03](ingestion.md#wf-03) through [PORT-18](ports-adapters.md#port-18). Human overrides survive because of [DM-10.2](domain.md#dm-10-2) and [DM-12.4](domain.md#dm-12-4); the handler adds no special casing.
- <a id="cmd-25-4"></a>**CMD-25.4 Idempotency.** As in [CMD-02.4](#cmd-02-4), using `run_key`.

Refs: [DM-17](domain.md#dm-17), [DM-03](domain.md#dm-03), [PORT-07](ports-adapters.md#port-07), [PORT-18](ports-adapters.md#port-18), [WF-03](ingestion.md#wf-03), [XC-06](overview.md#xc-06)

<a id="cmd-28"></a>
### CMD-28 SelectReprocessingPage

- <a id="cmd-28-1"></a>**CMD-28.1 Input.** `run_id`, `cursor` (null for the first page), `page_size` (default 500).
- <a id="cmd-28-2"></a>**CMD-28.2 Output.** `next_cursor`, `exhausted`, `to_process[]` (source item IDs whose stages were reset).
- <a id="cmd-28-3"></a>**CMD-28.3 Behaviour.** System-only, issued by [WF-03.2](ingestion.md#wf-03-2). The handler reads one page of the run's selection ([CMD-25.1](#cmd-25-1)) through `select_page` of [PORT-07](ports-adapters.md#port-07) ([PORT-07.3](ports-adapters.md#port-07-3)), resolves documents to their canonical source items, and in one transaction resets the run's requested stages and their dependants to `pending` ([ING-17.2](ingestion.md#ing-17-2)) and increments the run's `pending` counters ([CMD-26.3](#cmd-26-3)).
- <a id="cmd-28-4"></a>**CMD-28.4 Idempotency.** Key: (`run_id`, input cursor). Each reset records the `run_id` on the stage, and a stage already reset under this run is not reset again, so a replayed page has no further effect and returns the same `next_cursor`.

Refs: [DM-02](domain.md#dm-02), [DM-17](domain.md#dm-17), [PORT-07](ports-adapters.md#port-07), [WF-03](ingestion.md#wf-03), [ING-17](ingestion.md#ing-17)

## Queries

| ID | Query | Read model | Callers | Exposed via |
|---|---|---|---|---|
| [QRY-01](#qry-01) | SearchDocuments | [ADP-10.1](ports-adapters.md#adp-10-1) | member, reviewer, agent | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [MCP-03](mcp.md#mcp-03) |
| [QRY-02](#qry-02) | GetDocument | [ADP-10.5](ports-adapters.md#adp-10-5) | member, reviewer, agent | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [MCP-04](mcp.md#mcp-04) |
| [QRY-03](#qry-03) | GetDocumentContent | [ADP-10.5](ports-adapters.md#adp-10-5) | member, reviewer, agent (text only) | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [MCP-04](mcp.md#mcp-04) |
| [QRY-04](#qry-04) | GetThread | [ADP-10.5](ports-adapters.md#adp-10-5) | member, reviewer, agent | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [MCP-05](mcp.md#mcp-05) |
| [QRY-05](#qry-05) | ListClients | [ADP-10.5](ports-adapters.md#adp-10-5) | member, reviewer, agent | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [MCP-06](mcp.md#mcp-06) |
| [QRY-06](#qry-06) | ListMatters | [ADP-10.5](ports-adapters.md#adp-10-5) | member, reviewer, agent | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [MCP-06](mcp.md#mcp-06) |
| [QRY-07](#qry-07) | GetMatterOverview | [ADP-10.5](ports-adapters.md#adp-10-5) | member, reviewer, agent | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [MCP-07](mcp.md#mcp-07) |
| [QRY-08](#qry-08) | GetFacetCounts | [ADP-10.4](ports-adapters.md#adp-10-4) | member, reviewer, agent | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02) |
| [QRY-09](#qry-09) | ListReviewTasks | [ADP-10.5](ports-adapters.md#adp-10-5) | reviewer | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02) |
| [QRY-10](#qry-10) | GetIngestionRunProgress | [ADP-10.7](ports-adapters.md#adp-10-7) | admin | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [ADP-03](ports-adapters.md#adp-03) |
| [QRY-11](#qry-11) | ListIngestionFailures | [ADP-10.5](ports-adapters.md#adp-10-5) | admin | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [ADP-03](ports-adapters.md#adp-03) |
| [QRY-12](#qry-12) | GetTaxonomy | [ADP-10.5](ports-adapters.md#adp-10-5) | all roles | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [MCP-10](mcp.md#mcp-10) |
| [QRY-13](#qry-13) | FindSimilarDocuments | [ADP-10.6](ports-adapters.md#adp-10-6) | member, reviewer, agent | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02), [MCP-08](mcp.md#mcp-08) |
| [QRY-14](#qry-14) | GetAuditTrail | [ADP-10.5](ports-adapters.md#adp-10-5) | auditor | [ADP-01](ports-adapters.md#adp-01), [ADP-03](ports-adapters.md#adp-03) |
| [QRY-15](#qry-15) | ListPersons | [ADP-10.5](ports-adapters.md#adp-10-5) | member, reviewer, agent | [ADP-01](ports-adapters.md#adp-01), [ADP-02](ports-adapters.md#adp-02) |

<a id="qry-16"></a>
### QRY-16 Query handler contract

- <a id="qry-16-1"></a>**QRY-16.1 Shape.** A query is a frozen Pydantic model `<Name>Query` in `docstore.application.queries.<area>`, subclassing `Query[R]` imported from `docstore.ports` ([MOD-04.1](overview.md#mod-04-1)). Its handler returns a Pydantic DTO and receives the same `ExecutionContext` as commands ([CMD-26.1](#cmd-26-1)).
- <a id="qry-16-2"></a>**QRY-16.2 Read path only.** Queries read only through [PORT-11](ports-adapters.md#port-11), never through repositories, and never change aggregates. The one write is the access-audit entry ([QRY-16.7](#qry-16-7)), made by the query bus ([PORT-02.4](ports-adapters.md#port-02-4)), not the handler.
- <a id="qry-16-3"></a>**QRY-16.3 Permission predicate.** Every read-model call receives the principal's predicate inputs, built from [DM-16.6](domain.md#dm-16-6) to [DM-16.8](domain.md#dm-16-8). No query has an unfiltered variant. Role checks (Callers column) come first. A non-visible target yields `NotFound`, with the same response body and timing class as a missing ID ([XC-01](overview.md#xc-01)).
- <a id="qry-16-4"></a>**QRY-16.4 Derived visibility.** Assignments, family members, thread messages, locations, duplicates and counts that refer to non-visible matters or documents are omitted, without placeholders or "n hidden" counts. A location is non-visible when its `location_source_item_id` carries an accepted assignment to a non-visible matter.
- <a id="qry-16-5"></a>**QRY-16.5 Paging.** List queries use keyset cursors (default limit 50, maximum 500). Search uses offset paging up to rank 200 (default limit 20, or 10 for MCP; maximum 50). Each response carries `next_cursor` or `next_offset`.
- <a id="qry-16-6"></a>**QRY-16.6 Consistency.** Reads come from the same Postgres instance at read-committed isolation, so a committed command is visible to the next query; there is no projection lag ([ARCH-04](overview.md#arch-04)).
- <a id="qry-16-7"></a>**QRY-16.7 Access audit.** Every query call is audited, whatever its channel and outcome, with principal, acting agent, input, the applied filter and the returned IDs ([XC-08.3](overview.md#xc-08-3)).
- <a id="qry-16-8"></a>**QRY-16.8 Payload limits.** Snippets are at most 300 characters. A text response is at most 20,000 characters and includes the continuation range, which keeps MCP payloads bounded.

Refs: [ARCH-04](overview.md#arch-04), [PORT-02](ports-adapters.md#port-02), [PORT-11](ports-adapters.md#port-11), [XC-01](overview.md#xc-01)

## Queries: search and documents

<a id="qry-01"></a>
### QRY-01 SearchDocuments

- <a id="qry-01-1"></a>**QRY-01.1 Input.** `q` (optional text, at most 1,000 characters), `mode` ∈ {`hybrid`, `keyword`, `vector`} (default `hybrid`), `filters` {`client_ids[]`, `matter_ids[]`, `handler_ids[]`, `case_types[]`, `doc_types[]`, `tags[]`, `date_from`, `date_to` (on `document_date`), `source_ids[]`, `kinds[]`, `format_classes[]`, `senders[]`, `recipients[]`, `language`, `thread_id`, `unassigned_only`}, `collapse_near_duplicates` (default false), `include_facets` (default false), `limit`, `offset`, `candidates_per_leg` (default 100), `rrf_k` (default 60).
- <a id="qry-01-2"></a>**QRY-01.2 Output.** `hits[]` {`document_id`, `title_or_subject`, `kind`, `doc_type`, `client` and `matter` (visible ones only), `document_date`, `score`, `keyword_rank`, `vector_rank`, `best_chunk` {`ordinal`, `page_from`, `snippet` with highlights}}, `total_estimate`, `degraded` (true when the vector leg was skipped), `facets` (when requested, as [QRY-08](#qry-08)), `next_offset`.
- <a id="qry-01-3"></a>**QRY-01.3 Behaviour.** For `hybrid` and `vector`, the query text is embedded with `embed_query` of [PORT-12](ports-adapters.md#port-12). This is the only query-time egress ([XC-02](overview.md#xc-02)). If the provider fails or is rate-limited, the query falls back to `keyword` with `degraded = true` rather than failing. The single read-model statement ([ADR-03](decisions.md#adr-03)) applies the predicate and filters first, takes the top `candidates_per_leg` chunks per leg, fuses them with RRF, collapses to documents by best chunk, excludes `duplicate` documents, and optionally keeps one member per near-duplicate set. Without `q`, results are filtered documents ordered by `document_date` descending.
- <a id="qry-01-4"></a>**QRY-01.4 Targets.** The latency and relevance targets are [SC-05](../success-criteria.md#sc-05) and [SC-06](../success-criteria.md#sc-06).

Refs: [DM-03](domain.md#dm-03), [DM-05](domain.md#dm-05), [PORT-11](ports-adapters.md#port-11), [PORT-12](ports-adapters.md#port-12), [XC-01](overview.md#xc-01), [ADR-03](decisions.md#adr-03), [ADR-09](decisions.md#adr-09)

<a id="qry-02"></a>
### QRY-02 GetDocument

- <a id="qry-02-1"></a>**QRY-02.1 Input.** `document_id`, `include[]` ⊆ {`assignments`, `enrichment`, `provenance`, `family`, `duplicates`, `locations`, `history`} (default: all except `history`).
- <a id="qry-02-2"></a>**QRY-02.2 Output.** Identity, format, e-mail headers, dates and native metadata ([DM-03.1](domain.md#dm-03-1) to [DM-03.7](domain.md#dm-03-7)); `status`; accepted assignments with provenance and `proposed` ones for reviewers; enrichment fields with confidence, state and provenance; `family` (parent and children: ID, name, kind); `duplicates` (set kind, canonical, members); `locations` (display paths); `thread_id`; `chunk_count`; and, with `history`, superseded assignments and enrichment.
- <a id="qry-02-3"></a>**QRY-02.3 Rules.** Derived visibility follows [QRY-16.4](#qry-16-4). A `duplicate` document is returned with a pointer to its canonical document.

Refs: [DM-03](domain.md#dm-03), [DM-10](domain.md#dm-10), [DM-12](domain.md#dm-12), [PORT-11](ports-adapters.md#port-11), [XC-01](overview.md#xc-01)

<a id="qry-03"></a>
### QRY-03 GetDocumentContent

- <a id="qry-03-1"></a>**QRY-03.1 Input.** `document_id`, `form` ∈ {`text`, `original`}, and for text either `chunk_from` / `chunk_to` (ordinals) or `char_from` / `char_to`.
- <a id="qry-03-2"></a>**QRY-03.2 Output.** For `text`: `segments[]` {`ordinal`, `page_from`, `page_to`, `text`}, `truncated`, `next_range`, `total_chunks`. For `original`: a byte stream with `mime_type` and file name, read from [PORT-17](ports-adapters.md#port-17).
- <a id="qry-03-3"></a>**QRY-03.3 Rules.** `original` is available only to `human` principals; agents get text only. Text comes from the active chunk generation; for `duplicate` documents, which have no chunks, it comes from the text blob. Output is capped by [QRY-16.8](#qry-16-8).

Refs: [DM-03](domain.md#dm-03), [DM-05](domain.md#dm-05), [PORT-11](ports-adapters.md#port-11), [PORT-17](ports-adapters.md#port-17), [XC-01](overview.md#xc-01)

<a id="qry-04"></a>
### QRY-04 GetThread

- <a id="qry-04-1"></a>**QRY-04.1 Input.** `thread_id` or `document_id` (resolves to its thread), `include_snippets` (default true).
- <a id="qry-04-2"></a>**QRY-04.2 Output.** `thread_id`, `normalised_subject`, `messages[]` in `sent_at` order {`document_id`, `from_addr`, `to_addrs`, `sent_at`, `subject`, `snippet`, `attachments[]` {`document_id`, `attachment_name`, `doc_type`}}, plus `participants`.
- <a id="qry-04-3"></a>**QRY-04.3 Rules.** Only visible messages and attachments are returned ([QRY-16.4](#qry-16-4)). If none is visible, the result is `NotFound`.

Refs: [DM-06](domain.md#dm-06), [DM-03](domain.md#dm-03), [PORT-11](ports-adapters.md#port-11), [XC-01](overview.md#xc-01)

<a id="qry-13"></a>
### QRY-13 FindSimilarDocuments

- <a id="qry-13-1"></a>**QRY-13.1 Input.** `document_id`, `limit` (default 10, maximum 20), and the `filters` of [QRY-01.1](#qry-01-1).
- <a id="qry-13-2"></a>**QRY-13.2 Output.** `hits[]` as in [QRY-01.2](#qry-01-2), with `similarity` (cosine) instead of the rank fields.
- <a id="qry-13-3"></a>**QRY-13.3 Behaviour.** The query vector is the normalised mean of the source document's active chunk embeddings, computed in SQL. It makes no embedding API call. The handler runs a filtered kNN over chunks and collapses to documents. It excludes the document itself and the members of its duplicate sets ([DM-04.6](domain.md#dm-04-6)), which [QRY-02](#qry-02) lists. The source document must be visible.

Refs: [DM-05](domain.md#dm-05), [DM-04](domain.md#dm-04), [PORT-11](ports-adapters.md#port-11), [XC-01](overview.md#xc-01)

## Queries: organisation and browse

<a id="qry-05"></a>
### QRY-05 ListClients

- <a id="qry-05-1"></a>**QRY-05.1 Input.** `name_prefix`, `status`, `limit`, `cursor`.
- <a id="qry-05-2"></a>**QRY-05.2 Output.** `clients[]` {`client_id`, `pms_client_id`, `name`, `kind`, `visible_matter_count`, `visible_document_count`}.
- <a id="qry-05-3"></a>**QRY-05.3 Rules.** A client is listed only if the principal sees at least one of its matters ([DM-16.6](domain.md#dm-16-6)). Counts include visible items only.

Refs: [DM-07](domain.md#dm-07), [PORT-11](ports-adapters.md#port-11), [XC-01](overview.md#xc-01)

<a id="qry-06"></a>
### QRY-06 ListMatters

- <a id="qry-06-1"></a>**QRY-06.1 Input.** `client_id`, `responsible_lawyer_id`, `handler_id` (team member), `case_type`, `status`, `text` (matches code or title), `limit`, `cursor`.
- <a id="qry-06-2"></a>**QRY-06.2 Output.** `matters[]` {`matter_id`, `matter_code`, `title`, `client` {id, name}, `case_type`, `responsible_lawyer` {id, name}, `status`, `opened_on`, `document_count`, `last_document_date`}.
- <a id="qry-06-3"></a>**QRY-06.3 Rules.** Only visible matters are listed; a walled matter is absent, not greyed out.

Refs: [DM-09](domain.md#dm-09), [PORT-11](ports-adapters.md#port-11), [XC-01](overview.md#xc-01)

<a id="qry-07"></a>
### QRY-07 GetMatterOverview

- <a id="qry-07-1"></a>**QRY-07.1 Input.** `matter_id`, `recent_limit` (default 10).
- <a id="qry-07-2"></a>**QRY-07.2 Output.** The [QRY-06.2](#qry-06-2) row, plus `team[]` with roles; `people[]` (top 20 staff and external persons by document count, with party roles); `doc_type_breakdown`; `monthly_histogram` of `document_date`; `date_range`; `top_tags` (10); `recent_documents`; `thread_count`; and, for reviewers, `open_review_tasks`.
- <a id="qry-07-3"></a>**QRY-07.3 Rules.** A non-visible matter yields `NotFound`. All aggregates are computed over the matter's visible documents.

Refs: [DM-09](domain.md#dm-09), [DM-03](domain.md#dm-03), [PORT-11](ports-adapters.md#port-11), [XC-01](overview.md#xc-01)

<a id="qry-08"></a>
### QRY-08 GetFacetCounts

- <a id="qry-08-1"></a>**QRY-08.1 Input.** `q` and `filters` as in [QRY-01.1](#qry-01-1), and `facets[]` ⊆ {`client`, `matter`, `handler`, `case_type`, `doc_type`, `tag`, `year`, `source`, `kind`} (default all), `top_n` (default 20).
- <a id="qry-08-2"></a>**QRY-08.2 Output.** For each facet, `values[]` {`value`, `label`, `count`} plus `other_count`.
- <a id="qry-08-3"></a>**QRY-08.3 Rules.** Counts are of distinct documents, computed after the permission predicate. Without `q` they cover the full filtered set. With `q` they cover the fused candidate set of [QRY-01.3](#qry-01-3). Candidate tags and deprecated terms are never facet values ([TAX-03.3](domain.md#tax-03-3), [DM-14.4](domain.md#dm-14-4)).

Refs: [DM-03](domain.md#dm-03), [DM-10](domain.md#dm-10), [PORT-11](ports-adapters.md#port-11), [XC-01](overview.md#xc-01)

<a id="qry-12"></a>
### QRY-12 GetTaxonomy

- <a id="qry-12-1"></a>**QRY-12.1 Input.** `vocabularies[]` (default all), `include_candidates` (reviewer and admin only), `include_deprecated`.
- <a id="qry-12-2"></a>**QRY-12.2 Output.** `taxonomy_version`, and trees of {`code`, `labels`, `synonyms`, `status`, `replaced_by`, `children[]`}.
- <a id="qry-12-3"></a>**QRY-12.3 Rules.** Taxonomy is not matter-scoped, so the output carries no document counts; counts come from [QRY-08](#qry-08).

Refs: [DM-14](domain.md#dm-14), [TAX-01](domain.md#tax-01), [TAX-02](domain.md#tax-02), [TAX-03](domain.md#tax-03), [PORT-11](ports-adapters.md#port-11)

<a id="qry-15"></a>
### QRY-15 ListPersons

- <a id="qry-15-1"></a>**QRY-15.1 Input.** `kind` ∈ {`staff`, `external`}, `staff_role`, `matter_id`, `name_prefix`, `limit`, `cursor`.
- <a id="qry-15-2"></a>**QRY-15.2 Output.** `persons[]` {`person_id`, `kind`, `name`, `initials`, `emails`, `staff_role`, `matters[]` {`matter_id`, `role`}}.
- <a id="qry-15-3"></a>**QRY-15.3 Rules.** Staff are always listed (firm directory), but their `matters` show visible matters only. External persons are listed only when linked to a visible document or matter.

Refs: [DM-08](domain.md#dm-08), [PORT-11](ports-adapters.md#port-11), [XC-01](overview.md#xc-01)

## Queries: review, operations and audit

<a id="qry-09"></a>
### QRY-09 ListReviewTasks

- <a id="qry-09-1"></a>**QRY-09.1 Input.** `state` (default `open`), `reason[]`, `matter_id`, `assignee_id`, `document_id`, `limit`, `cursor`.
- <a id="qry-09-2"></a>**QRY-09.2 Output.** `tasks[]` {`task_id`, `reason`, `priority`, `subject`, `document` {id, title, doc_type, snippet}, `candidates[]` with provenance, `current_value`, `comments`, `opened_by`, `opened_at`, `assignee_id`}. Ordered by priority, then `opened_at`.
- <a id="qry-09-3"></a>**QRY-09.3 Rules.** Walls apply to reviewers: tasks on documents the reviewer cannot see are excluded ([DM-16.2](domain.md#dm-16-2)).

Refs: [DM-13](domain.md#dm-13), [PORT-11](ports-adapters.md#port-11)

<a id="qry-10"></a>
### QRY-10 GetIngestionRunProgress

- <a id="qry-10-1"></a>**QRY-10.1 Input.** `run_id` (or `source_id` for the latest run).
- <a id="qry-10-2"></a>**QRY-10.2 Output.** `state`, `kind`, `started_at`, `elapsed`; per stage the counts by status ([DM-17.2](domain.md#dm-17-2)); `discovered_total` and `discovery_exhausted`; `bytes_fetched`; `throughput_docs_per_min` (last 5 minutes and since start); `eta`; `queue_depths` per queue ([WF-04](ingestion.md#wf-04)) from [PORT-18](ports-adapters.md#port-18); `recent_failures` (5); `as_of`.
- <a id="qry-10-3"></a>**QRY-10.3 Rules.** The figures come from the counters maintained by [ING-16](ingestion.md#ing-16) and must be no more than 10 s stale ([SC-10](../success-criteria.md#sc-10)). ETA is the remaining pending items divided by the 5-minute throughput, and is null until 100 items are done.

Refs: [DM-17](domain.md#dm-17), [PORT-11](ports-adapters.md#port-11), [PORT-18](ports-adapters.md#port-18), [ING-16](ingestion.md#ing-16)

<a id="qry-11"></a>
### QRY-11 ListIngestionFailures

- <a id="qry-11-1"></a>**QRY-11.1 Input.** `run_id`, `stage`, `error_class`, `status` ∈ {`failed`, `quarantined`}, `limit`, `cursor`.
- <a id="qry-11-2"></a>**QRY-11.2 Output.** `items[]` {`source_item_id`, `source_key`, `display_path`, `document_id`, `stage`, `status`, `error_class`, `last_error`, `attempts`, `updated_at`}, plus totals by error class.
- <a id="qry-11-3"></a>**QRY-11.3 Rules.** Admin only. Display paths are shown unfiltered, because admins operate sources; document content is not exposed here.

Refs: [DM-02](domain.md#dm-02), [DM-17](domain.md#dm-17), [PORT-11](ports-adapters.md#port-11)

<a id="qry-14"></a>
### QRY-14 GetAuditTrail

- <a id="qry-14-1"></a>**QRY-14.1 Input.** Any of `document_id` (matched against `target_ids` and `result_ids`), `principal_id`, `acting_agent_id`, `matter_id` (matched against `matter_ids`), `action` (event type, command or query name), `from`, `to`; plus `limit` and `cursor`.
- <a id="qry-14-2"></a>**QRY-14.2 Output.** `entries[]` with the columns of [XC-08.1](overview.md#xc-08-1); `result_ids` only on request.
- <a id="qry-14-3"></a>**QRY-14.3 Rules.** `auditor` role only. Auditors see entries for walled matters, because wall compliance is what they verify. Reading the audit trail is itself audited.

Refs: [XC-08](overview.md#xc-08), [PORT-11](ports-adapters.md#port-11)
