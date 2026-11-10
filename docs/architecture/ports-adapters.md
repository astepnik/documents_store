# Ports and adapters

This file defines the ports of the hexagon ([PORT-01](#port-01) to [PORT-20](#port-20)) and the adapters that implement or drive them ([ADP-01](#adp-01) to [ADP-22](#adp-22)), with the PoC technology for each. The hexagon diagram, layer rules and module paths are in [ARCH-01](overview.md#arch-01), [ARCH-03](overview.md#arch-03), [MOD-04](overview.md#mod-04), [MOD-05](overview.md#mod-05) and [MOD-06](overview.md#mod-06). The commands and queries that use the ports are catalogued in [application.md](application.md), starting at [CMD-01](application.md#cmd-01) and [QRY-01](application.md#qry-01).

Interface sketches are Python 3.12 `typing.Protocol`s, simplified. Domain types (`Document`, `Principal`, `AccessScope` and so on) are those of [domain.md](domain.md). Port DTOs are frozen dataclasses, or frozen Pydantic v2 models when an inbound adapter serialises them.

## Catalogue

| Port | Direction | PoC adapter | Test double ([ADP-22](#adp-22)) |
|---|---|---|---|
| [PORT-01](#port-01) CommandHandler | inbound | driven by [ADP-01](#adp-01) to [ADP-05](#adp-05) | direct handler calls |
| [PORT-02](#port-02) QueryHandler | inbound | driven by [ADP-01](#adp-01) to [ADP-04](#adp-04) | direct handler calls |
| [PORT-03](#port-03) Authenticator | outbound | [ADP-07](#adp-07) | `FakeAuthenticator` |
| [PORT-04](#port-04) UnitOfWork | outbound | [ADP-08](#adp-08) | `InMemoryUnitOfWork` |
| [PORT-05](#port-05) to [PORT-09](#port-09) repositories | outbound | [ADP-08](#adp-08) | in-memory repositories |
| [PORT-10](#port-10) SearchIndexWriter | outbound | [ADP-09](#adp-09) | `InMemoryIndex` |
| [PORT-11](#port-11) ReadModel | outbound | [ADP-10](#adp-10) | `InMemoryReadModel` |
| [PORT-12](#port-12) Embedder | outbound | [ADP-12](#adp-12) | `HashEmbedder` |
| [PORT-13](#port-13) LlmClient | outbound | [ADP-13](#adp-13) | `ScriptedLlm` |
| [PORT-14](#port-14) TextExtractor | outbound | [ADP-14](#adp-14), [ADP-15](#adp-15) | `FixtureExtractor` |
| [PORT-15](#port-15) OcrEngine | outbound | [ADP-16](#adp-16) | `FixtureOcr` |
| [PORT-16](#port-16) SourceConnector | outbound | [ADP-17](#adp-17), [ADP-18](#adp-18), [ADP-19](#adp-19) | `ListConnector` |
| [PORT-17](#port-17) BlobStore | outbound | [ADP-20](#adp-20) | `MemoryBlobStore` |
| [PORT-18](#port-18) WorkflowEngine | outbound | [ADP-06](#adp-06) | `ImmediateWorkflowEngine` |
| [PORT-19](#port-19) EventPublisher | outbound | [ADP-21](#adp-21) | `RecordingPublisher` |

---

## Port conventions

<a id="port-20"></a>
### PORT-20 Port design conventions

These rules apply to every port below, so individual ports do not repeat them.

- <a id="port-20-1"></a>**PORT-20.1 Synchronous.** All ports are synchronous Python calls. DBOS steps, Typer commands and the query path are synchronous in the PoC. Adapters that wrap async libraries (msgraph-sdk, the `mcp` SDK) bridge internally and never leak coroutines through a port.
- <a id="port-20-2"></a>**PORT-20.2 Typed errors.** Outbound adapters raise only the error types of [XC-05.2](overview.md#xc-05-2) (`TransientError`, `BackpressureError`, `PermanentError`) plus the application errors of [PORT-01.3](#port-01-3). Library exceptions never cross a port. Unknown exceptions count as transient, as [XC-05.2](overview.md#xc-05-2) states.
- <a id="port-20-3"></a>**PORT-20.3 No I/O types in signatures.** Port signatures use domain types, IDs, `pathlib.Path`, `bytes` and `BinaryIO`. They never use SQLAlchemy rows, `httpx` responses or DBOS handles, so [ARCH-03](overview.md#arch-03) holds for `docstore.ports`.
- <a id="port-20-4"></a>**PORT-20.4 Version reporting.** Every adapter that produces derived data reports the version stamp it applied (parser, OCR, model, prompt), as listed in [XC-06.1](overview.md#xc-06-1). The caller stores the stamp; the adapter never decides staleness.
- <a id="port-20-5"></a>**PORT-20.5 Large payloads go by reference.** Bytes larger than 1 MiB pass as a `Path` in a step-scoped temporary directory, or as a `BlobRef` ([PORT-17](#port-17)), never as an in-memory `bytes` value. Workflow checkpoints therefore never contain content ([ADP-05.4](#adp-05-4)).
- <a id="port-20-6"></a>**PORT-20.6 Contract suites.** Each outbound port has one abstract pytest contract suite in `tests/contract/` that both the fake and the real adapter must pass ([XC-07.3](overview.md#xc-07-3)). The behaviours listed as sub-items of a port below are the cases of that suite.

Refs: [XC-05](overview.md#xc-05), [XC-06](overview.md#xc-06), [XC-07](overview.md#xc-07), [ARCH-03](overview.md#arch-03)

---

## Inbound ports

<a id="port-01"></a>
### PORT-01 CommandHandler (inbound)

```python
class CommandBus(Protocol):
    def execute(self, cmd: Command[R], ctx: ExecutionContext) -> R: ...

class CommandHandler(Protocol[C, R]):
    def __call__(self, cmd: C, ctx: ExecutionContext) -> R: ...
```

- <a id="port-01-1"></a>**PORT-01.1 Dispatch.** Inbound adapters call only `CommandBus.execute`. The bus resolves the handler by the command's type from the bootstrap registry ([ARCH-04.6](overview.md#arch-04-6)). Command, result and `ExecutionContext` shapes are owned by [CMD-26.1](application.md#cmd-26-1).
- <a id="port-01-2"></a>**PORT-01.2 Context is built by the adapter.** The inbound adapter builds the `ExecutionContext` from the authenticated principal ([PORT-03](#port-03)), the channel and its request ID. [ADP-05](#adp-05) adds `run_id` and `workflow_id`. Handlers never read ambient state for identity.
- <a id="port-01-3"></a>**PORT-01.3 Bus middleware and errors.** In order: bind the log context ([XC-03.2](overview.md#xc-03-2)), check the channel allowlist ([PORT-01.4](#port-01-4)), call the handler. The bus does not write the audit entry of a successful command; the unit of work does ([PORT-04.5](#port-04-5)). When the handler raises and [CMD-26.10](application.md#cmd-26-10) requires an entry, the bus writes it with the error outcome in its own short transaction. The bus never opens a transaction around the handler; handlers do that through [PORT-04](#port-04). Handler errors ([CMD-26.8](application.md#cmd-26-8)) pass through unchanged. The only error added at the port layer is `AuthenticationError`, raised by [PORT-03](#port-03). Each inbound adapter maps them to its protocol ([ADP-01.2](#adp-01-2), [ADP-03.3](#adp-03-3), [MCP-01.8](mcp.md#mcp-01-8)).
- <a id="port-01-4"></a>**PORT-01.4 Channel allowlist.** Each inbound adapter builds its bus with the set of command types it may issue, as a second guard next to the role check of [CMD-26.2](application.md#cmd-26-2). The MCP bus allows only [CMD-20](application.md#cmd-20) ([MCP-02.6](mcp.md#mcp-02-6)). The HTTP, UI and CLI buses exclude the pipeline commands [CMD-04](application.md#cmd-04) to [CMD-16](application.md#cmd-16), [CMD-27](application.md#cmd-27) and [CMD-28](application.md#cmd-28), which only [ADP-05](#adp-05) issues. [CMD-29](application.md#cmd-29) is allowed on the HTTP and UI buses only.

Refs: [ARCH-04](overview.md#arch-04), [CMD-01](application.md#cmd-01), [ADP-01](#adp-01), [ADP-02](#adp-02), [ADP-03](#adp-03), [ADP-04](#adp-04), [ADP-05](#adp-05)

<a id="port-02"></a>
### PORT-02 QueryHandler (inbound)

```python
class QueryBus(Protocol):
    def ask(self, qry: Query[R], ctx: ExecutionContext) -> R: ...

class QueryHandler(Protocol[Q, R]):
    def __call__(self, qry: Q, ctx: ExecutionContext) -> R: ...
```

- <a id="port-02-1"></a>**PORT-02.1 Dispatch.** The bus resolves handlers by query type ([ARCH-04.6](overview.md#arch-04-6)). Query shape, read-only path and paging rules are owned by [QRY-16](application.md#qry-16).
- <a id="port-02-2"></a>**PORT-02.2 Read DTOs are Pydantic models.** Result types are frozen Pydantic v2 models, so the HTTP adapter exposes them in OpenAPI and the MCP adapter derives tool `outputSchema` from the same class ([MCP-01.5](mcp.md#mcp-01-5)). One DTO, two protocols, which gives the parity measured by [SC-11.5](../success-criteria.md#sc-11-5).
- <a id="port-02-3"></a>**PORT-02.3 Cursor encoding.** The keyset cursors and search offsets of [QRY-16.5](application.md#qry-16-5) leave the bus as opaque strings: base64url JSON with the query hash, the position and the principal ID, signed with HMAC-SHA256 under a server key. A cursor presented by another principal, or for another query, fails with `ValidationFailed`, so cursors cannot carry results across principals or replicas.
- <a id="port-02-4"></a>**PORT-02.4 Access audit.** The bus is the only writer of access entries ([QRY-16.2](application.md#qry-16-2)). For every query call ([QRY-16.7](application.md#qry-16-7)), including failed ones, it writes the access entry through [PORT-19.3](#port-19-3) before returning, fail-closed ([XC-08.3](overview.md#xc-08-3)). Channel, acting agent and any adapter-supplied call details ([MCP-02.7](mcp.md#mcp-02-7)) come from the `ExecutionContext`.

Refs: [ARCH-04](overview.md#arch-04), [QRY-01](application.md#qry-01), [ADP-01](#adp-01), [ADP-02](#adp-02), [ADP-03](#adp-03), [ADP-04](#adp-04)

<a id="port-03"></a>
### PORT-03 Authenticator

```python
class Authenticator(Protocol):
    def authenticate(self, credential: BearerToken, channel: Channel) -> Principal: ...
```

- <a id="port-03-1"></a>**PORT-03.1 No anonymous access.** A missing, unknown, expired or channel-mismatched credential, or an inactive principal, raises `AuthenticationError`. There is no default principal ([XC-01.1](overview.md#xc-01-1)).
- <a id="port-03-2"></a>**PORT-03.2 Identity only.** The authenticator maps a credential to a `principal_id` and loads that principal from policy through [PORT-06](#port-06). Kind, roles, groups and, for agents, `acts_for_principal_id` all come from the imported policy ([DM-16.1](domain.md#dm-16-1)), so a token can never carry more rights than the policy gives. The adapter puts the principal into the `ExecutionContext` ([PORT-01.2](#port-01-2)).
- <a id="port-03-3"></a>**PORT-03.3 Log context.** The inbound adapter also binds `principal_id` to the log context ([XC-03.2](overview.md#xc-03-2)) for the duration of the request.
- <a id="port-03-4"></a>**PORT-03.4 Production swap.** An OAuth 2.1 resource-server adapter (JWT validation against Entra ID or Keycloak) replaces [ADP-07](#adp-07) behind this same port ([ADR-10.5](decisions.md#adr-10-5)).

Refs: [DM-16](domain.md#dm-16), [ADP-07](#adp-07), [XC-01](overview.md#xc-01)

---

## Outbound ports: persistence

<a id="port-04"></a>
### PORT-04 UnitOfWork

```python
class UnitOfWork(Protocol):
    documents: DocumentRepository
    organisation: OrganisationRepository
    ingestion: IngestionRepository
    reviews: ReviewTaskRepository
    classification: ClassificationRepository
    index: SearchIndexWriter                # same transaction
    def __enter__(self) -> Self: ...
    def __exit__(self, *exc) -> None: ...   # rolls back unless committed
    def commit(self) -> None: ...
    def rollback(self) -> None: ...

UnitOfWorkFactory = Callable[[], UnitOfWork]
```

- <a id="port-04-1"></a>**PORT-04.1 Scope.** One `with uow_factory() as uow:` block is one database transaction ([ARCH-06.1](overview.md#arch-06-1)). Repositories are usable only inside it. Using one after exit raises `RuntimeError`.
- <a id="port-04-2"></a>**PORT-04.2 Event collection.** The unit of work tracks every aggregate loaded or added. After a successful `commit`, it drains their `pending_events` and hands them to [PORT-19](#port-19), following [ARCH-05.2](overview.md#arch-05-2).
- <a id="port-04-3"></a>**PORT-04.3 Optimistic locking.** `save` of a stale aggregate raises `ConcurrencyConflict`, a subclass of `TransientError`, so the step or request is retried ([ARCH-06.2](overview.md#arch-06-2)).
- <a id="port-04-4"></a>**PORT-04.4 Uniqueness.** A unique-key violation on a natural key ([ARCH-06.4](overview.md#arch-06-4)) is surfaced as `DuplicateKey(key)` so that idempotent handlers can treat "already exists" as success.
- <a id="port-04-5"></a>**PORT-04.5 Command audit.** When the `ExecutionContext` principal is `human` or `agent` ([CMD-26.10](application.md#cmd-26-10)), `commit` first inserts the command's audit entry ([XC-08.1](overview.md#xc-08-1)) in the same transaction, with `target_ids` taken from the aggregates saved. A handler that commits several units of work (per-file imports, [CMD-26.3](application.md#cmd-26-3)) gets one entry per commit.

Refs: [ARCH-06](overview.md#arch-06), [ADP-08](#adp-08), [PORT-19](#port-19)

<a id="port-05"></a>
### PORT-05 DocumentRepository

```python
class DocumentRepository(Protocol):
    def get(self, id: DocumentId, *, with_chunks: bool = False) -> Document: ...
    def find_by_sha256(self, sha256: Sha256) -> Document | None: ...
    def find_by_email_identity(self, key: EmailIdentityKey) -> Document | None: ...
    def add(self, doc: Document) -> None: ...
    def save(self, doc: Document) -> None: ...
    def replace_chunks(self, id: DocumentId, generation: ChunkGeneration, chunks: Sequence[Chunk]) -> None: ...
    def chunks_pending_embedding(self, id: DocumentId, model: EmbeddingModelRef, limit: int) -> list[Chunk]: ...
    def store_vectors(self, vectors: Sequence[ChunkVector], model: EmbeddingModelRef) -> int: ...
    def get_duplicate_set(self, id: DuplicateSetId) -> DuplicateSet: ...
    def near_duplicate_candidates(self, bands: Sequence[int], limit: int) -> list[DocumentId]: ...
    def save_duplicate_set(self, s: DuplicateSet) -> None: ...
    def find_thread(self, message_ids: Sequence[str], subject_key: str) -> Thread | None: ...
    def save_thread(self, t: Thread) -> None: ...
```

- <a id="port-05-1"></a>**PORT-05.1 Chunks are bulk-loaded separately.** `get` loads the Document aggregate ([DM-03](domain.md#dm-03)) without chunks by default. Chunks of a large document (thousands of rows with vectors) are written by `replace_chunks`, which deletes and inserts the chunks of one chunk generation of a document in one statement batch, and are read only through `with_chunks=True` or the embedding helpers.
- <a id="port-05-2"></a>**PORT-05.2 Vector writes are idempotent.** `store_vectors` updates only chunks whose `embedding` is null or stamped with another model, and returns the number of rows changed. Repeating a batch after a crash changes nothing ([ING-18](ingestion.md#ing-18)).
- <a id="port-05-3"></a>**PORT-05.3 Near-duplicate lookup.** `near_duplicate_candidates` returns documents that share at least one locality-sensitive-hashing band with the given signature. The signature algorithm belongs to [CMD-09](application.md#cmd-09); the repository only stores and matches bands.

Refs: [DM-03](domain.md#dm-03), [DM-04](domain.md#dm-04), [DM-05](domain.md#dm-05), [DM-06](domain.md#dm-06), [ADP-08](#adp-08)

<a id="port-06"></a>
### PORT-06 OrganisationRepository

```python
class OrganisationRepository(Protocol):
    def get_client(self, id: ClientId) -> Client: ...
    def get_matter(self, id: MatterId) -> Matter: ...
    def find_matter_by_code(self, code: str) -> Matter | None: ...
    def clients_by_email_domain(self, domain: str) -> list[Client]: ...
    def persons_by_email(self, address: str) -> list[Person]: ...
    def upsert_clients(self, items: Sequence[Client]) -> UpsertStats: ...
    def upsert_persons(self, items: Sequence[Person]) -> UpsertStats: ...
    def upsert_matters(self, items: Sequence[Matter]) -> UpsertStats: ...
    def access_policy(self, at: date) -> AccessPolicy: ...
    def replace_access_policy(self, policy: AccessPolicy) -> None: ...
```

- <a id="port-06-1"></a>**PORT-06.1 Upserts are keyed by PMS identifier.** Re-importing the same CSV is a no-op that reports `unchanged` counts ([CMD-21](application.md#cmd-21)).
- <a id="port-06-2"></a>**PORT-06.2 Lookups for rules.** The `find_*` and `*_by_*` methods are the only lookups [CMD-10](application.md#cmd-10) needs. They are backed by indexes on matter code, lower-cased e-mail domain and lower-cased address.
- <a id="port-06-3"></a>**PORT-06.3 Policy snapshot.** `access_policy(at)` returns the grants and walls in effect on the given date ([DM-16](domain.md#dm-16)).

Refs: [DM-07](domain.md#dm-07), [DM-08](domain.md#dm-08), [DM-09](domain.md#dm-09), [DM-16](domain.md#dm-16), [ADP-08](#adp-08)

<a id="port-07"></a>
### PORT-07 IngestionRepository

```python
class IngestionRepository(Protocol):
    def get_source(self, id: SourceId) -> Source: ...
    def save_source(self, s: Source) -> None: ...
    def upsert_items(self, source: SourceId, run: RunId, items: Sequence[DiscoveredItem]) -> UpsertStats: ...
    def get_item(self, id: SourceItemId) -> SourceItem: ...
    def find_item(self, source: SourceId, key: SourceKey) -> SourceItem | None: ...
    def save_item(self, item: SourceItem) -> None: ...
    def get_run(self, id: RunId) -> IngestionRun: ...
    def find_run_by_key(self, run_key: str) -> IngestionRun | None: ...
    def save_run(self, run: IngestionRun) -> None: ...
    def add_stage_counts(self, run: RunId, deltas: Mapping[tuple[Stage, Outcome], int]) -> None: ...
    def count_selection(self, selection: ReprocessSelection) -> SelectionEstimate: ...
    def select_page(self, selection: ReprocessSelection, after: SourceItemId | None, limit: int) -> list[SourceItemId]: ...
```

- <a id="port-07-1"></a>**PORT-07.1 Fingerprint-aware upsert.** `upsert_items` inserts new items, updates items whose fingerprint changed (resetting their stage status), and leaves unchanged items untouched. It returns `new`, `changed` and `unchanged` counts for [ING-01](ingestion.md#ing-01).
- <a id="port-07-2"></a>**PORT-07.2 Counters bypass the aggregate lock.** `add_stage_counts` is an atomic `UPDATE … SET n = n + :d` on the run's counter rows. It does not load or version-check the IngestionRun aggregate, so thousands of concurrent steps do not conflict on it. What is counted is defined by [ING-16](ingestion.md#ing-16).
- <a id="port-07-3"></a>**PORT-07.3 Reprocessing selection.** `count_selection` returns the number of selected documents and the inputs for the work estimate of [CMD-25.2](application.md#cmd-25-2); `select_page` returns one keyset page of selected source items for [CMD-28](application.md#cmd-28) (SelectRunPage), for both reprocess runs and enrich runs. Both read on the `docstore_app` connection, return only IDs and counts, and are never exposed to a principal; the selection fields are those of [CMD-25.1](application.md#cmd-25-1).

Refs: [DM-01](domain.md#dm-01), [DM-02](domain.md#dm-02), [DM-17](domain.md#dm-17), [ADP-08](#adp-08)

<a id="port-08"></a>
### PORT-08 ReviewTaskRepository

```python
class ReviewTaskRepository(Protocol):
    def get(self, id: ReviewTaskId) -> ReviewTask: ...
    def find_open(self, document: DocumentId, subject: ReviewSubject) -> ReviewTask | None: ...
    def add(self, task: ReviewTask) -> None: ...
    def save(self, task: ReviewTask) -> None: ...
```

- <a id="port-08-1"></a>**PORT-08.1** `find_open` lets [CMD-12](application.md#cmd-12) and [CMD-20](application.md#cmd-20) attach to an existing open task for the same document and subject (for example, the matter assignment) instead of opening a second one.

Refs: [DM-13](domain.md#dm-13), [ADP-08](#adp-08)

<a id="port-09"></a>
### PORT-09 ClassificationRepository

```python
class ClassificationRepository(Protocol):
    def taxonomy(self, kind: TaxonomyKind, *, include_candidates: bool = False) -> Taxonomy: ...
    def get_term(self, kind: TaxonomyKind, code: str) -> TaxonomyTerm | None: ...
    def save_term(self, term: TaxonomyTerm) -> None: ...
    def published_ruleset(self) -> RuleSet: ...
    def get_ruleset(self, version: int) -> RuleSet: ...
    def add_ruleset_version(self, rs: RuleSet) -> None: ...
```

- <a id="port-09-1"></a>**PORT-09.1 Immutable rule-set versions.** `add_ruleset_version` inserts only. Updating a published version raises `Conflict`, which enforces [DM-15](domain.md#dm-15) at the storage level as well.
- <a id="port-09-2"></a>**PORT-09.2 Cached reads.** `taxonomy` and `published_ruleset` are read on every enrichment and rule step. The adapter caches them per process and invalidates the cache on the term or rule-set events, through an in-process subscriber ([ARCH-05.3](overview.md#arch-05-3)).

Refs: [DM-14](domain.md#dm-14), [DM-15](domain.md#dm-15), [ADP-08](#adp-08)

---

## Outbound ports: search

<a id="port-10"></a>
### PORT-10 SearchIndexWriter

```python
class SearchIndexWriter(Protocol):
    def index_document(self, id: DocumentId, generation: ChunkGeneration, keys: FilterKeys) -> IndexStats: ...
    def refresh_filter_keys(self, id: DocumentId, keys: FilterKeys) -> None: ...
    def refresh_enrichment_keys(self, id: DocumentId, keys: EnrichmentFilterKeys) -> None: ...
    def remove_document(self, id: DocumentId) -> None: ...

@dataclass(frozen=True)
class FilterKeys: ...   # one attribute per denormalised filter key of the domain, same names

@dataclass(frozen=True)
class EnrichmentFilterKeys: ...   # the document-level keys only; no access, client or matter keys
```

- <a id="port-10-4"></a>**PORT-10.4 Filter keys.** `FilterKeys` has exactly the fields, names and types of [DM-05.3](domain.md#dm-05-3); the port adds none. `EnrichmentFilterKeys` has exactly the document-level group of [DM-05.3](domain.md#dm-05-3), so access keys, `client_ids` and `matter_ids` cannot be passed to it.
- <a id="port-10-5"></a>**PORT-10.5 Stage-2 write path.** `refresh_enrichment_keys` writes only the document-level keys and never touches `chunks`. It is the only method the stage-2 metadata mode of [CMD-15.5](application.md#cmd-15-5) calls. The contract suite of this port ([XC-07.3](overview.md#xc-07-3)) asserts, against [ADP-09](#adp-09) and the fake of [ADP-22.2](#adp-22-2), that after it runs, the access keys, `client_ids`, `matter_ids` and every chunk row of the document are byte-identical, which checks [DM-10.8](domain.md#dm-10-8) and [SC-16.2](../success-criteria.md#sc-16-2) structurally.

- <a id="port-10-1"></a>**PORT-10.1 Same transaction.** The writer is reached as `uow.index` ([PORT-04](#port-04)), so filter keys and access keys commit atomically with the assignment or policy change that caused them ([XC-01.2](overview.md#xc-01-2)).
- <a id="port-10-2"></a>**PORT-10.2 Generation swap.** `index_document` writes keys onto every chunk of the given generation, removes chunks of older generations from the index, and only then sets `status = indexed` and `active_chunk_generation` ([DM-03.10](domain.md#dm-03-10)). A document whose chunks lack vectors for the active model is refused with `PermanentError("not_embedded")`.
- <a id="port-10-3"></a>**PORT-10.3 Idempotent.** Calling `index_document` twice with the same arguments leaves identical rows.

Refs: [DM-05](domain.md#dm-05), [ADP-09](#adp-09)

<a id="port-11"></a>
### PORT-11 ReadModel

```python
class ReadModel(Protocol):
    def search(self, req: SearchRequest, scope: AccessScope) -> SearchPage: ...
    def facets(self, f: SearchFilters, text: str | None, scope: AccessScope) -> FacetCounts: ...
    def document(self, id: DocumentId, scope: AccessScope) -> DocumentView | None: ...
    def document_text(self, id: DocumentId, chunks: range, scope: AccessScope) -> TextSlice | None: ...
    def original(self, id: DocumentId, scope: AccessScope) -> BlobRef | None: ...
    def thread(self, id: ThreadId, scope: AccessScope, page: PageReq) -> ThreadView | None: ...
    def similar(self, id: DocumentId, k: int, f: SearchFilters, scope: AccessScope) -> list[DocumentHit]: ...
    def clients(self, f: ClientFilter, scope: AccessScope, page: PageReq) -> Page[ClientRow]: ...
    def matters(self, f: MatterFilter, scope: AccessScope, page: PageReq) -> Page[MatterRow]: ...
    def matter_overview(self, id: MatterId, scope: AccessScope) -> MatterOverview | None: ...
    def persons(self, f: PersonFilter, scope: AccessScope, page: PageReq) -> Page[PersonRow]: ...
    def review_tasks(self, f: ReviewFilter, scope: AccessScope, page: PageReq) -> Page[ReviewRow]: ...
    def run_progress(self, id: RunId) -> RunProgress | None: ...
    def run_failures(self, id: RunId, scope: AccessScope, page: PageReq) -> Page[FailureRow]: ...
    def taxonomy(self) -> TaxonomyView: ...
    def audit(self, f: AuditFilter, page: PageReq) -> Page[AuditRow]: ...

@dataclass(frozen=True)
class SearchRequest:
    text: str
    query_vector: Sequence[float] | None   # None means keyword-only (mode or degraded)
    model: EmbeddingModelRef | None        # must equal the indexed model
    mode: Literal["hybrid", "keyword", "vector"]
    filters: SearchFilters
    page: PageReq
```

- <a id="port-11-1"></a>**PORT-11.1 Scope is mandatory.** Every method that can return a document, chunk, thread, matter, client or person takes an `AccessScope` positional argument without a default. `AccessScope` holds the predicate inputs of [QRY-16.3](application.md#qry-16-3), built per [XC-01.3](overview.md#xc-01-3). No method exists that reads document-derived rows without it. `run_progress`, `taxonomy` and `audit` return no document content; their handlers restrict them by role ([DM-16.2](domain.md#dm-16-2)).
- <a id="port-11-2"></a>**PORT-11.2 Absent means invisible or nonexistent.** Methods return `None` or an empty page, and the handler raises `NotFound`. The adapter never distinguishes the two cases ([XC-01.5](overview.md#xc-01-5)).
- <a id="port-11-3"></a>**PORT-11.3 Query vector from outside.** The read model never calls the embedding endpoint. [QRY-01](application.md#qry-01) obtains `query_vector` from [PORT-12](#port-12) and decides the degraded keyword-only fallback.
- <a id="port-11-4"></a>**PORT-11.4 Model match.** If `req.model` differs from the model stamped on the indexed chunks, the vector leg returns nothing and the page is marked `degraded=true` rather than comparing vectors from different spaces ([ADR-16.2](decisions.md#adr-16-2)).
- <a id="port-11-5"></a>**PORT-11.5 Snippets.** Snippets are taken from the best-matching visible chunk, with match markers `«…»`, within the length limit of [QRY-16.8](application.md#qry-16-8).

Refs: [XC-01](overview.md#xc-01), [ADP-10](#adp-10), [ARCH-04](overview.md#arch-04), [QRY-01](application.md#qry-01)

---

## Outbound ports: AI services

<a id="port-12"></a>
### PORT-12 Embedder

```python
class Embedder(Protocol):
    @property
    def model(self) -> EmbeddingModelRef: ...       # model_id, weights_digest, dimension
    @property
    def limits(self) -> EmbeddingLimits: ...        # max_texts, max_tokens_per_request; from configuration
    def embed_documents(self, texts: Sequence[str], token_counts: Sequence[int]) -> EmbeddingBatch: ...
    def embed_query(self, text: str) -> list[float]: ...
    def status(self) -> ModelEndpointStatus: ...    # read-only; never raises BackpressureError

@dataclass(frozen=True)
class EmbeddingBatch:
    vectors: list[list[float]]   # same order and length as texts
    model: EmbeddingModelRef
    tokens: int
```

- <a id="port-12-1"></a>**PORT-12.1 Document and query roles.** Documents and queries are separate calls so that models with asymmetric encoding get a query-side instruction (Qwen3-Embedding: `embedding.QUERY_INSTRUCTION`, [ADP-12.1](#adp-12-1)). Models without one (BGE-M3) treat both the same.
- <a id="port-12-2"></a>**PORT-12.2 Batch limits are the caller's job.** [CMD-14](application.md#cmd-14) packs batches within `limits`, using the token counts that the chunker stored ([DM-05](domain.md#dm-05)). A batch over the limits raises `PermanentError("batch_too_large")`. The adapter never truncates text silently, and the endpoint is never relied on to do so.
- <a id="port-12-3"></a>**PORT-12.3 Back-pressure signal.** A saturated, warming or restarting endpoint raises `BackpressureError(retry_after_s)`, taking the value from `Retry-After` when the endpoint sends it. The `embed` queue cap ([WF-04.8](ingestion.md#wf-04-8)) is the primary throttle, and this signal is the feedback path.
- <a id="port-12-4"></a>**PORT-12.4 Dimension check.** The adapter verifies that every vector it returns has `model.dimension` entries, after the truncation of [ADP-12.3](#adp-12-3), and raises `PermanentError("dimension_mismatch")` otherwise. Startup also checks the configured dimension against the column ([XC-04.4](overview.md#xc-04-4)).
- <a id="port-12-5"></a>**PORT-12.5 Production swap.** **Retired:** a local embedding model is now the default rather than a production swap. Replaced by [PORT-12.6](#port-12-6).
- <a id="port-12-6"></a>**PORT-12.6 Endpoint-only binding.** The port's real adapter is [ADP-12](#adp-12), against any OpenAI-compatible `/v1/embeddings` endpoint: vLLM by default ([DEP-09](overview.md#dep-09)), or llama.cpp or Ollama by configuration.
- <a id="port-12-7"></a>**PORT-12.7 Status.** `status()` returns a `ModelEndpointStatus`: served model name and the digest it carries, readiness, the last warm-up result, the `/metrics` gauges of [XC-03.7](overview.md#xc-03-7), and whether the served name matches `models.lock` ([DEP-10.1](overview.md#dep-10-1)). It is read by [QRY-17](application.md#qry-17) only and changes no state. Changing the runtime or the model is configuration only ([XC-04](overview.md#xc-04)). Changing the model weights requires a re-embed ([ING-17.5](ingestion.md#ing-17-5)).

Refs: [ADP-12](#adp-12), [DEP-09](overview.md#dep-09), [ADR-16](decisions.md#adr-16)

<a id="port-13"></a>
### PORT-13 LlmClient

```python
class LlmClient(Protocol):
    def complete_structured(self, req: StructuredRequest) -> StructuredResponse: ...
    def status(self) -> ModelEndpointStatus: ...    # as PORT-12.7

@dataclass(frozen=True)
class StructuredRequest:
    system: str
    user: str                    # contains the delimited document excerpt
    schema_name: str
    json_schema: Mapping[str, Any]
    max_output_tokens: int
    temperature: float = 0.0
    seed: int | None = None      # set from LLM-01.4

@dataclass(frozen=True)
class StructuredResponse:
    data: Mapping[str, Any]      # parsed and schema-valid
    raw_text: str
    model_id: str                # as served, e.g. "gpt-oss-120b@sha256:…"
    model_digest: str | None     # weights digest from the served name (LLM-01.3), for provenance
    prompt_tokens: int
    completion_tokens: int
    latency_ms: int
    finish_reason: str
```

- <a id="port-13-1"></a>**PORT-13.1 Validated output.** The adapter parses `raw_text` as JSON and validates it against `json_schema` (`jsonschema`, Draft 2020-12). Invalid JSON, a schema violation or `finish_reason == "length"` raises `InvalidLlmOutput(raw_text, errors)`, a `PermanentError` subtype that [LLM-05](ingestion.md#llm-05) uses for its repair retry.
- <a id="port-13-2"></a>**PORT-13.2 No tools, no streaming.** The port has no tool-calling or streaming surface, which keeps the safeguards of [LLM-06](ingestion.md#llm-06) structural.
- <a id="port-13-3"></a>**PORT-13.3 Provenance fields.** `model_id`, `model_digest` and the request parameters are what [LLM-04](ingestion.md#llm-04) stamps. `model_digest` is the weights digest carried in the served model name ([LLM-01.3](ingestion.md#llm-01-3)). The reported request parameters include the merged `llm.EXTRA_BODY` (for example `reasoning_effort`, [ADP-13.1](#adp-13-1)), so they are part of the stamped `params`. The adapter reports them; it does not build provenance.
- <a id="port-13-4"></a>**PORT-13.4 Status.** `status()` has the contract of [PORT-12.7](#port-12-7) for the `llm` role.
- <a id="port-13-5"></a>**PORT-13.5 Seed.** `seed` defaults to `None` in the DTO; the enrichment step sets it from the request parameters of [LLM-01.4](ingestion.md#llm-01-4).

Refs: [ADP-13](#adp-13), [LLM-01](ingestion.md#llm-01), [ADR-15](decisions.md#adr-15)

---

## Outbound ports: content

<a id="port-14"></a>
### PORT-14 TextExtractor

```python
class TextExtractor(Protocol):
    def detect(self, path: Path, filename_hint: str | None) -> DetectedFormat: ...   # mime, format_class
    def extract(self, path: Path, fmt: DetectedFormat, workdir: Path) -> Extraction: ...

@dataclass(frozen=True)
class Extraction:
    text: str
    has_text_layer: bool
    page_count: int | None
    chars_per_page: list[int] | None
    native_metadata: Mapping[str, str]    # normalised keys: title, author, created, modified, ...
    email: EmailHeaders | None            # message_id, in_reply_to, references, from, to, cc, date, subject
    children: list[EmbeddedChild]         # name, mime, path (in workdir), ordinal
    parser_version: str
    warnings: list[str]
```

- <a id="port-14-1"></a>**PORT-14.1 Composite routing.** `CompositeExtractor` implements the port. It detects the format with [ADP-14](#adp-14) and routes `message/rfc822` and `application/vnd.ms-outlook` to [ADP-15](#adp-15), and everything else to [ADP-14](#adp-14). The routing table is data in the adapter, not logic in the application.
- <a id="port-14-2"></a>**PORT-14.2 Text-layer flag.** `has_text_layer` is false for images, and for PDFs whose pages carry no extractable characters (from Tika's `pdf:charsPerPage`). The "usable text" threshold that triggers OCR is applied by [CMD-07.3](application.md#cmd-07-3), using `chars_per_page`.
- <a id="port-14-3"></a>**PORT-14.3 Children by path.** Attachments and embedded files are written into `workdir` and returned as paths ([PORT-20.5](#port-20-5)). [CMD-06](application.md#cmd-06) stores them in [PORT-17](#port-17) and creates child items.
- <a id="port-14-4"></a>**PORT-14.4 Failure classes.** Encrypted, DRM-protected, corrupt and unsupported inputs raise `PermanentError` with reason codes `encrypted`, `corrupt` or `unsupported_format`. Tika timeouts and child restarts raise `TransientError` ([XC-05.1](overview.md#xc-05-1)).

Refs: [ADP-14](#adp-14), [ADP-15](#adp-15), [ADR-07](decisions.md#adr-07)

<a id="port-15"></a>
### PORT-15 OcrEngine

```python
class OcrEngine(Protocol):
    def ocr(self, path: Path, languages: Sequence[str], workdir: Path) -> OcrResult: ...

@dataclass(frozen=True)
class OcrResult:
    text: str
    pages: list[PageOcr]           # page_no, chars, mean_confidence (0-100 or None)
    searchable_pdf: Path | None    # PDF with a text layer, stored by the caller
    ocr_version: str               # e.g. "ocrmypdf-16/tesseract-5/eng+pol"
```

- <a id="port-15-1"></a>**PORT-15.1 Only where needed.** Pages that already have text are kept as they are (skip-text semantics), so mixed PDFs are OCRed only on their image pages.
- <a id="port-15-2"></a>**PORT-15.2 Inputs.** The engine accepts PDF, TIFF (including multi-page), PNG and JPEG. Other formats raise `PermanentError("unsupported_format")`.
- <a id="port-15-3"></a>**PORT-15.3 Empty result.** An OCR run that produces no text is not an error at the port. The caller decides, with [XC-05.1](overview.md#xc-05-1) (`empty text after OCR`).

Refs: [ADP-16](#adp-16), [ADR-07](decisions.md#adr-07)

<a id="port-16"></a>
### PORT-16 SourceConnector

```python
class SourceConnector(Protocol):
    kind: SourceKind                                  # "filesystem" | "pst" | "sharepoint"
    def check(self, config: SourceConfig) -> ConnectivityReport: ...
    def list_page(self, config: SourceConfig, cursor: str | None, limit: int) -> ItemPage: ...
    def fetch(self, config: SourceConfig, key: SourceKey, workdir: Path) -> FetchedItem: ...

@dataclass(frozen=True)
class DiscoveredItem:
    key: SourceKey                 # stable across runs and connector modes
    fingerprint: str               # changes if and only if content or metadata changed
    display_path: str
    size: int | None
    modified_at: datetime | None
    source_metadata: Mapping[str, str]   # site, library, columns, mailbox owner, folder, ...
    deleted: bool = False          # tombstone from delta sources
    excluded_by: str | None = None # exclude rule that matched; reported, not dropped

@dataclass(frozen=True)
class ItemPage:
    items: list[DiscoveredItem]
    next_cursor: str | None        # None means discovery is complete
```

- <a id="port-16-1"></a>**PORT-16.1 Resumable paging.** `list_page(cursor)` is deterministic for an unchanged source. Repeating a page after a crash returns the same items or a superset, and never skips. The cursor is an opaque JSON string that [CMD-04](application.md#cmd-04) persists after each page.
- <a id="port-16-2"></a>**PORT-16.2 Stable keys.** `key` is unique within the source and never derived from mutable display names when a stable ID exists. It is the idempotency anchor of [DM-02](domain.md#dm-02).
- <a id="port-16-3"></a>**PORT-16.3 Fetch writes to disk.** `fetch` writes the item's bytes to `workdir` and returns its path, the detected filename and any metadata only available at fetch time. It never returns bytes in memory ([PORT-20.5](#port-20-5)).
- <a id="port-16-4"></a>**PORT-16.4 Read-only.** Connectors never modify, move or mark items at the source ([SCOPE-07](../README.md#scope-07)).
- <a id="port-16-5"></a>**PORT-16.5 Registry.** The bootstrap registers one connector per `kind`. [CMD-01](application.md#cmd-01) and [CMD-04](application.md#cmd-04) look connectors up by the source's kind.

Refs: [DM-01](domain.md#dm-01), [ADP-17](#adp-17), [ADP-18](#adp-18), [ADP-19](#adp-19)

<a id="port-17"></a>
### PORT-17 BlobStore

```python
class BlobStore(Protocol):
    def put_file(self, path: Path) -> BlobRef: ...           # BlobRef: sha256, size
    def put_text(self, text: str) -> BlobRef: ...            # UTF-8 extracted text
    def open(self, sha256: Sha256) -> BinaryIO: ...
    def exists(self, sha256: Sha256) -> bool: ...
    def local_path(self, sha256: Sha256) -> Path | None: ... # zero-copy for OCR and Tika when local
```

- <a id="port-17-1"></a>**PORT-17.1 Content addressing.** The key is the SHA-256 of the bytes, computed while copying. `put_*` of existing content is a no-op that returns the same `BlobRef`.
- <a id="port-17-2"></a>**PORT-17.2 Atomic writes.** A blob is visible only after it is complete: the adapter writes a temporary file, runs `fsync`, then renames it. A crash leaves no partial blob under a final key.
- <a id="port-17-3"></a>**PORT-17.3 No deletes in the PoC.** The port has no delete. Purging is the demo purge of [XC-02.6](overview.md#xc-02-6).

Refs: [ADP-20](#adp-20), [ADR-11](decisions.md#adr-11)

---

## Outbound ports: runtime

<a id="port-18"></a>
### PORT-18 WorkflowEngine

```python
class WorkflowEngine(Protocol):
    def start(self, spec: WorkflowSpec) -> WorkflowStatus: ...   # name, workflow_id, queue, args
    def cancel(self, workflow_id: str) -> None: ...
    def resume(self, workflow_id: str) -> None: ...
    def status(self, workflow_id: str) -> WorkflowStatus | None: ...
    def list(self, id_prefix: str, states: set[WorkflowState] | None, limit: int) -> list[WorkflowStatus]: ...

@dataclass(frozen=True)
class WorkflowStatus:
    workflow_id: str
    name: str
    state: WorkflowState      # pending | enqueued | running | succeeded | error | cancelled | retries_exceeded
    created_at: datetime
    updated_at: datetime
    error: str | None
```

- <a id="port-18-1"></a>**PORT-18.1 Deterministic IDs.** `start` with an existing `workflow_id` returns the existing workflow's status and starts nothing. The ID formats are defined in [WF-01](ingestion.md#wf-01), [WF-02](ingestion.md#wf-02), [WF-06](ingestion.md#wf-06) and [WF-07](ingestion.md#wf-07).
- <a id="port-18-2"></a>**PORT-18.2 Names, not callables.** Workflows are addressed by registered name (`ingestion_run`, `document_pipeline`, `reprocessing_run`, `enrichment_run`, `document_enrichment`), so the application never imports workflow code, as [ARCH-08](overview.md#arch-08) requires.
- <a id="port-18-3"></a>**PORT-18.3 Control semantics.** What pause and resume mean for a run is defined in [WF-05](ingestion.md#wf-05). The port offers only the engine primitives cancel and resume.

Refs: [ADP-06](#adp-06), [ARCH-08](overview.md#arch-08), [ADR-04](decisions.md#adr-04)

<a id="port-19"></a>
### PORT-19 EventPublisher

```python
class EventPublisher(Protocol):
    def publish(self, events: Sequence[DomainEvent]) -> None: ...
    def subscribe(self, event_type: type[DomainEvent], handler: Callable[[DomainEvent], None]) -> None: ...
    def record_access(self, entry: AccessAuditEntry) -> None: ...
```

- <a id="port-19-1"></a>**PORT-19.1 Post-commit only.** `publish` is called by [PORT-04](#port-04) after commit. Delivery is synchronous and in process ([ARCH-05.2](overview.md#arch-05-2)).
- <a id="port-19-2"></a>**PORT-19.2 Subscription at bootstrap.** `subscribe` is called only by the composition root ([ARCH-07](overview.md#arch-07)). Handler order is registration order, and the audit subscriber is always first.
- <a id="port-19-3"></a>**PORT-19.3 Access audit.** `record_access` writes one access entry ([XC-08.3](overview.md#xc-08-3)) in its own short transaction and raises if the write fails, which makes the calling query fail closed.

Refs: [ADP-21](#adp-21), [ARCH-05](overview.md#arch-05), [ADR-13](decisions.md#adr-13)

---

## Inbound adapters

<a id="adp-01"></a>
### ADP-01 FastAPI HTTP API

FastAPI app at `/api/v1` in the api service ([DEP-03](overview.md#dep-03)), module [MOD-05.1](overview.md#mod-05-1). Request and response bodies are the command DTOs and read DTOs ([PORT-02.2](#port-02-2)).

- <a id="adp-01-1"></a>**ADP-01.1 Auth dependency.** A FastAPI dependency reads `Authorization: Bearer`, calls [PORT-03](#port-03) with channel `http` and builds the `ExecutionContext` ([PORT-01.2](#port-01-2)). Every route depends on it except `/healthz` and `/readyz`.
- <a id="adp-01-2"></a>**ADP-01.2 Error mapping.** `AuthenticationError` → 401 with `WWW-Authenticate: Bearer`. `Forbidden` → 403. `NotFound` → 404 with the same body for invisible and nonexistent items. `ValidationFailed` → 422. `Conflict` and `InvariantViolation` → 409. `ExternalRateLimited` → 429 with `Retry-After`. `ExternalBackpressure` → 503 with `Retry-After`. `ExternalTransient` and `ExternalPermanent` → 503. Anything else → 500 with `request_id` only, never a stack trace or SQL.
- <a id="adp-01-3"></a>**ADP-01.3 Request ID.** Middleware accepts `X-Request-ID` or generates a UUIDv7, binds it to the log context ([XC-03.2](overview.md#xc-03-2)) and echoes it in the response.

Route map (writes are `POST`, `PUT` or `PATCH`; pipeline commands are not routed, per [PORT-01.4](#port-01-4)):

| ID | Routes | Maps to |
|---|---|---|
| <a id="adp-01-4"></a>ADP-01.4 | `POST /sources` | [CMD-01](application.md#cmd-01) |
| <a id="adp-01-5"></a>ADP-01.5 | `POST /runs`; `POST /runs/{id}/pause`, `/resume`, `/cancel`, `/throttle` | [CMD-02](application.md#cmd-02), [CMD-03](application.md#cmd-03) |
| <a id="adp-01-6"></a>ADP-01.6 | `GET /runs/{id}`; `GET /runs/{id}/failures` | [QRY-10](application.md#qry-10), [QRY-11](application.md#qry-11) |
| <a id="adp-01-7"></a>ADP-01.7 | `POST /reprocessing` | [CMD-25](application.md#cmd-25) |
| <a id="adp-01-8"></a>ADP-01.8 | `POST /search`; `POST /search/facets` | [QRY-01](application.md#qry-01), [QRY-08](application.md#qry-08) |
| <a id="adp-01-9"></a>ADP-01.9 | `GET /documents/{id}`; `GET /documents/{id}/text?from=&to=`; `GET /documents/{id}/original` (streamed); `GET /documents/{id}/similar` | [QRY-02](application.md#qry-02), [QRY-03](application.md#qry-03), [QRY-13](application.md#qry-13) |
| <a id="adp-01-10"></a>ADP-01.10 | `GET /threads/{id}` | [QRY-04](application.md#qry-04) |
| <a id="adp-01-11"></a>ADP-01.11 | `GET /clients`; `GET /matters`; `GET /matters/{id}/overview`; `GET /persons` | [QRY-05](application.md#qry-05), [QRY-06](application.md#qry-06), [QRY-07](application.md#qry-07), [QRY-15](application.md#qry-15) |
| <a id="adp-01-12"></a>ADP-01.12 | `PUT /documents/{id}/assignments/{role}`; `PATCH /documents/{id}/enrichment`; `POST /documents/{id}/flags` | [CMD-17](application.md#cmd-17), [CMD-18](application.md#cmd-18), [CMD-20](application.md#cmd-20) |
| <a id="adp-01-13"></a>ADP-01.13 | `GET /review-tasks`; `POST /review-tasks/{id}/resolve` | [QRY-09](application.md#qry-09), [CMD-19](application.md#cmd-19) |
| <a id="adp-01-14"></a>ADP-01.14 | `POST /imports/reference-data`; `POST /imports/access-policies` (multipart CSV) | [CMD-21](application.md#cmd-21), [CMD-22](application.md#cmd-22) |
| <a id="adp-01-15"></a>ADP-01.15 | `GET /taxonomy`; `PUT /taxonomy/{kind}/terms/{code}`; `POST /rulesets` | [QRY-12](application.md#qry-12), [CMD-23](application.md#cmd-23), [CMD-24](application.md#cmd-24) |
| <a id="adp-01-16"></a>ADP-01.16 | `GET /audit` | [QRY-14](application.md#qry-14) |
| <a id="adp-01-17"></a>ADP-01.17 | `GET /models/status` | [QRY-17](application.md#qry-17) |

Refs: [PORT-01](#port-01), [PORT-02](#port-02), [PORT-03](#port-03), [DEP-03](overview.md#dep-03), [ADR-02](decisions.md#adr-02)

<a id="adp-02"></a>
### ADP-02 Web UI (Jinja2 + HTMX)

Server-rendered pages served by the api process under `/` ([DEP-03](overview.md#dep-03)), module [MOD-05.2](overview.md#mod-05-2). The UI calls the buses in process, not the HTTP API.

| ID | Page | Uses |
|---|---|---|
| <a id="adp-02-1"></a>ADP-02.1 | `/` search with filter sidebar and facet counts | [QRY-01](application.md#qry-01), [QRY-08](application.md#qry-08), [QRY-12](application.md#qry-12) |
| <a id="adp-02-2"></a>ADP-02.2 | `/documents/{id}`: metadata, assignments and enrichment with provenance, text pane, original download, similar documents, override forms | [QRY-02](application.md#qry-02), [QRY-03](application.md#qry-03), [QRY-13](application.md#qry-13), [CMD-17](application.md#cmd-17), [CMD-18](application.md#cmd-18), [CMD-20](application.md#cmd-20) |
| <a id="adp-02-3"></a>ADP-02.3 | `/threads/{id}` | [QRY-04](application.md#qry-04) |
| <a id="adp-02-4"></a>ADP-02.4 | `/matters`, `/matters/{id}` | [QRY-06](application.md#qry-06), [QRY-07](application.md#qry-07) |
| <a id="adp-02-5"></a>ADP-02.5 | `/review`: queue with candidate comparison and one-click resolve | [QRY-09](application.md#qry-09), [CMD-19](application.md#cmd-19) |
| <a id="adp-02-6"></a>ADP-02.6 | `/runs`, `/runs/{id}`: progress partial polled by HTMX ([XC-03.4](overview.md#xc-03-4)), failure list, stage-2 backlog and lag of the companion enrich run, pause, resume, cancel and throttle controls | [QRY-10](application.md#qry-10), [QRY-11](application.md#qry-11), [CMD-03](application.md#cmd-03) |

- <a id="adp-02-7"></a>**ADP-02.7 PoC login.** `/login` accepts a bearer token, validates it through [PORT-03](#port-03) (channel `ui`) and sets a signed, `HttpOnly`, `SameSite=Strict` session cookie that holds the principal ID and expires after 8 hours. There is no password store.
- <a id="adp-02-8"></a>**ADP-02.8 CSRF and escaping.** State-changing HTMX requests carry a per-session CSRF token header. Jinja2 autoescape is on, and extracted text and review comments are rendered as escaped text only, because document content is untrusted.

Refs: [PORT-01](#port-01), [PORT-02](#port-02), [DEP-03](overview.md#dep-03), [ADR-14](decisions.md#adr-14)

<a id="adp-03"></a>
### ADP-03 CLI (Typer)

`docstore` console script, module [MOD-05.3](overview.md#mod-05-3), run on the host via `uv run docstore` or in compose as the `cli` service ([DEP-01.7](overview.md#dep-01-7)). The command groups are listed in [MOD-05.3](overview.md#mod-05-3).

- <a id="adp-03-1"></a>**ADP-03.1 Principal.** The CLI authenticates with `DOCSTORE_CLI_TOKEN` through [PORT-03](#port-03) (channel `cli`). There is no bypass for local use; administrative commands need the `admin` role ([CMD-26.2](application.md#cmd-26-2)).
- <a id="adp-03-2"></a>**ADP-03.2 Key mappings.** `source add` → [CMD-01](application.md#cmd-01); `run start|pause|resume|cancel` → [CMD-02](application.md#cmd-02), [CMD-03](application.md#cmd-03); `enrich pause|resume|throttle|rerun` → [CMD-03](application.md#cmd-03), [CMD-25](application.md#cmd-25); `models status` → [QRY-17](application.md#qry-17); `models verify` is not a CLI command but the host-side operator script `deploy/models/verify.sh`, which runs the one-shot `model-verify` job of [DEP-10.3](overview.md#dep-10-3); it is outside the hexagon and calls no handler; `run watch|failures` → [QRY-10](application.md#qry-10), [QRY-11](application.md#qry-11); `import refdata|acl` → [CMD-21](application.md#cmd-21), [CMD-22](application.md#cmd-22); `taxonomy upsert|show` → [CMD-23](application.md#cmd-23), [QRY-12](application.md#qry-12); `rules publish` → [CMD-24](application.md#cmd-24); `review list|resolve` → [QRY-09](application.md#qry-09), [CMD-19](application.md#cmd-19); `reprocess` → [CMD-25](application.md#cmd-25); `eval` runs the harness of [F-20](../features.md#f-20). Tokens are issued outside the CLI ([ADP-07.3](#adp-07-3)).
- <a id="adp-03-3"></a>**ADP-03.3 Exit codes.** 0 success; 2 `ValidationFailed`; 3 `NotFound`; 4 `AuthenticationError` or `Forbidden`; 5 `Conflict` or `InvariantViolation`; 6 `External*`; 1 anything else. `--json` prints the DTO for scripting.
- <a id="adp-03-4"></a>**ADP-03.4 Starting runs.** `run start` executes [CMD-02](application.md#cmd-02) in the CLI process. The workflow is enqueued through [ADP-06](#adp-06) and executed by the worker service, so the CLI may exit immediately or follow with `--watch`.

Refs: [PORT-01](#port-01), [PORT-02](#port-02), [ADR-14](decisions.md#adr-14)

<a id="adp-04"></a>
### ADP-04 MCP server adapter

The implementation of the server specified in [mcp.md](mcp.md). Module [MOD-05.4](overview.md#mod-05-4), process [DEP-04](overview.md#dep-04).

- <a id="adp-04-1"></a>**ADP-04.1 SDK and app.** `mcp.server.fastmcp.FastMCP("docstore", stateless_http=True, json_response=True)`, served through its Streamable HTTP ASGI app, mounted in a Starlette app at `/mcp` under uvicorn, next to `/healthz` and `/readyz`.
- <a id="adp-04-2"></a>**ADP-04.2 Auth middleware.** A Starlette middleware in front of the MCP app validates `Origin`, extracts the bearer token, calls [PORT-03](#port-03) with channel `mcp`, stores the principal in a request-scoped `contextvars` variable from which tool functions build the `ExecutionContext` ([PORT-01.2](#port-01-2)), and rejects with 401 before the SDK sees the request. The behaviour is specified in [MCP-02](mcp.md#mcp-02).
- <a id="adp-04-3"></a>**ADP-04.3 Tool functions are thin.** Each `@mcp.tool()` function in `adapters/inbound/mcp/tools.py` builds one query or command DTO from its typed arguments, calls the bus ([PORT-01](#port-01) or [PORT-02](#port-02)), and shapes the DTO into the tool result. It contains no SQL, no permission logic and no business rules.
- <a id="adp-04-4"></a>**ADP-04.4 Schemas from types.** Tool input schemas come from the annotated Pydantic argument models, and output schemas come from the read DTOs ([PORT-02.2](#port-02-2)). Both are generated, never hand-written.
- <a id="adp-04-5"></a>**ADP-04.5 Restricted bus.** The adapter is wired with a command bus whose allowlist contains only [CMD-20](application.md#cmd-20) ([PORT-01.4](#port-01-4)).

Refs: [PORT-01](#port-01), [PORT-02](#port-02), [PORT-03](#port-03), [MCP-01](mcp.md#mcp-01), [DEP-04](overview.md#dep-04), [ADR-10](decisions.md#adr-10)

<a id="adp-05"></a>
### ADP-05 DBOS ingestion worker

Hosts the DBOS workflows and queues in the worker service ([DEP-05](overview.md#dep-05)), module [MOD-05.5](overview.md#mod-05-5). The workflow definitions are [WF-01](ingestion.md#wf-01) to [WF-03](ingestion.md#wf-03), [WF-06](ingestion.md#wf-06) and [WF-07](ingestion.md#wf-07), and the queue topology is [WF-04](ingestion.md#wf-04).

- <a id="adp-05-1"></a>**ADP-05.1 Launch.** `bootstrap.worker` builds the container ([ARCH-07](overview.md#arch-07)), configures `DBOS(config={"name": "docstore", "system_database_url": settings.db.DBOS_URL, "executor_id": settings.worker.EXECUTOR_ID})`, imports the workflow modules so that decorators register, declares the queues, and calls `DBOS.launch()`. Launch recovers pending workflows of this executor ([WF-05](ingestion.md#wf-05)).
- <a id="adp-05-2"></a>**ADP-05.2 One step, one command.** Each `@DBOS.step()` function executes exactly one command through [PORT-01](#port-01) and returns its result. The `ExecutionContext` carries the configured `service` principal with role `worker` ([DM-16.2](domain.md#dm-16-2)), the run ID, and `DBOS.workflow_id`. Workflow functions only sequence steps, branch on step results, enqueue children and sleep. They hold no business rules ([ARCH-08](overview.md#arch-08)).
- <a id="adp-05-3"></a>**ADP-05.3 Determinism.** Workflow functions contain no I/O, clock reads or randomness outside steps. The current time is read inside a step, so it is checkpointed, and waits use durable `DBOS.sleep`, so that replay after a crash takes the same path.
- <a id="adp-05-4"></a>**ADP-05.4 Small checkpoints.** Step return values are only IDs, hashes, counts and status codes. Text, vectors and file bytes never enter the DBOS system database, which keeps it small at 100 GB scale ([PORT-20.5](#port-20-5)).
- <a id="adp-05-5"></a>**ADP-05.5 Error translation.** The adapter maps the error classes of [XC-05.1](overview.md#xc-05-1) onto DBOS step retry settings for `transient`, durable sleeps for `backpressure`, and an immediate call to [CMD-16](application.md#cmd-16) for `permanent` and `poison`. What happens to the item then is [ING-15](ingestion.md#ing-15).

Refs: [PORT-01](#port-01), [WF-01](ingestion.md#wf-01), [WF-02](ingestion.md#wf-02), [WF-03](ingestion.md#wf-03), [WF-04](ingestion.md#wf-04), [WF-06](ingestion.md#wf-06), [WF-07](ingestion.md#wf-07), [DEP-05](overview.md#dep-05), [ADR-04](decisions.md#adr-04)

---

## Outbound adapters

<a id="adp-06"></a>
### ADP-06 DBOS workflow engine client

Implements [PORT-18](#port-18), module [MOD-06.8](overview.md#mod-06-8).

- <a id="adp-06-1"></a>**ADP-06.1 Two modes.** In the worker process, the adapter uses the in-process API (`DBOS.start_workflow` on the named queue, `DBOS.cancel_workflow`, `DBOS.resume_workflow`, `DBOS.list_workflows`). In the api, mcp and cli processes, which do not launch DBOS, it uses `DBOSClient(system_database_url)` with `enqueue(options, *args)`, where `options` carries the workflow name, queue name and `workflow_id`, and uses the client's cancel, resume, retrieve and list calls.
- <a id="adp-06-2"></a>**ADP-06.2 Status mapping.** DBOS statuses (`PENDING`, `ENQUEUED`, `SUCCESS`, `ERROR`, `CANCELLED`, `MAX_RECOVERY_ATTEMPTS_EXCEEDED`) map one to one onto `WorkflowState`.
- <a id="adp-06-3"></a>**ADP-06.3 Duplicate start.** Enqueueing an existing `workflow_id` is a no-op in DBOS, and the adapter returns the existing status ([PORT-18.1](#port-18-1)).

Refs: [PORT-18](#port-18), [ADR-04](decisions.md#adr-04)

<a id="adp-07"></a>
### ADP-07 Static token authenticator

Implements [PORT-03](#port-03) for the PoC, module [MOD-06.10](overview.md#mod-06-10).

- <a id="adp-07-1"></a>**ADP-07.1 Token file.** JSON at `DOCSTORE_AUTH__TOKENS_FILE` (a Docker secret, [XC-02.5](overview.md#xc-02-5)). Each entry has `token_sha256`, `principal_id`, `channels` (subset of `http`, `ui`, `cli`, `mcp`), `label` (for example, the MCP client it was issued to) and optional `expires_at`. Only hashes are stored. Principal kind and roles are not in the file ([PORT-03.2](#port-03-2)).
- <a id="adp-07-2"></a>**ADP-07.2 Verification.** The adapter computes SHA-256 of the presented token, looks the hash up, compares with `hmac.compare_digest`, then checks the channel and expiry, and loads the principal from policy ([PORT-03.2](#port-03-2)).
- <a id="adp-07-3"></a>**ADP-07.3 Issuing.** `uv run python -m docstore.adapters.outbound.auth.issue_token --principal <id> --channel mcp --label <client>` generates `secrets.token_urlsafe(32)`, prints it once and appends the hash entry. Revocation means removing the entry and restarting the process: the file is read at startup only ([ADR-10.3](decisions.md#adr-10-3)).

Refs: [PORT-03](#port-03), [ADR-10](decisions.md#adr-10)

<a id="adp-08"></a>
### ADP-08 Postgres unit of work and repositories

Implements [PORT-04](#port-04) to [PORT-09](#port-09), module [MOD-06.1](overview.md#mod-06-1). Schema and migrations are in [MOD-08](overview.md#mod-08).

- <a id="adp-08-1"></a>**ADP-08.1 Engine.** SQLAlchemy 2.0 Core `Engine` on `postgresql+psycopg://` as role `docstore_app` ([DEP-02.3](overview.md#dep-02-3)), `QueuePool` sized by `DOCSTORE_DB__POOL_SIZE`, `READ COMMITTED`, `statement_timeout` of 30 s set per connection.
- <a id="adp-08-2"></a>**ADP-08.2 Hand-written mappers.** Each repository maps rows to domain objects explicitly. There is no ORM and no lazy loading, so the domain stays free of persistence types ([ARCH-03](overview.md#arch-03)).
- <a id="adp-08-3"></a>**ADP-08.3 Optimistic lock.** `save` runs `UPDATE … SET …, version = version + 1 WHERE id = :id AND version = :v`. If the row count is 0, it raises `ConcurrencyConflict` ([PORT-04.3](#port-04-3)).
- <a id="adp-08-4"></a>**ADP-08.4 Bulk paths.** `upsert_items`, `replace_chunks` and `store_vectors` use `executemany` with `INSERT … ON CONFLICT` or `UPDATE … FROM (VALUES …)` in batches of 500. Vectors are bound as pgvector `halfvec` through `pgvector.psycopg.register_vector`.
- <a id="adp-08-5"></a>**ADP-08.5 Repository to table map.**

  | Repository | Main tables |
  |---|---|
  | [PORT-05](#port-05) | `documents`, `document_sources`, `chunks`, `duplicate_sets`, `duplicate_members`, `minhash_bands`, `threads`, `assignments`, `enrichments` |
  | [PORT-06](#port-06) | `clients`, `client_domains`, `persons`, `person_emails`, `matters`, `matter_team`, `principals`, `groups`, `matter_grants`, `walls` |
  | [PORT-07](#port-07) | `sources`, `source_items`, `ingestion_runs`, `run_stage_counters` |
  | [PORT-08](#port-08) | `review_tasks`, `review_candidates` |
  | [PORT-09](#port-09) | `taxonomy_terms`, `rule_sets` |

Refs: [PORT-04](#port-04), [PORT-05](#port-05), [PORT-06](#port-06), [PORT-07](#port-07), [PORT-08](#port-08), [PORT-09](#port-09), [MOD-08](overview.md#mod-08), [ADR-03](decisions.md#adr-03)

<a id="adp-09"></a>
### ADP-09 Postgres search index writer

Implements [PORT-10](#port-10), module [MOD-06.2](overview.md#mod-06-2). It runs on the unit-of-work connection.

- <a id="adp-09-1"></a>**ADP-09.1 Indexes are declarative.** The BM25 index (`USING bm25 (id, text, …) WITH (key_field='id')`) and the HNSW index on `chunks.embedding` are created by migrations ([MOD-08](overview.md#mod-08)) with the parameters of [ADR-12.2](decisions.md#adr-12-2). Postgres maintains them transactionally on row change, so the writer issues no index-specific calls.
- <a id="adp-09-2"></a>**ADP-09.2 What the writer does.** `index_document` runs one `UPDATE chunks SET <chunk-level keys of [DM-05.3](domain.md#dm-05-3)> = … WHERE document_id = :id AND generation = :g`, upserts the document-level keys ([ADP-09.5](#adp-09-5)), deletes the chunks of other generations of the document, and sets `status = indexed` and `active_chunk_generation` ([DM-03.10](domain.md#dm-03-10)). `refresh_filter_keys` runs the same writes without the generation swap; it serves the access refresh of [CMD-12](application.md#cmd-12). `refresh_enrichment_keys` runs only the upsert of [ADP-09.5](#adp-09-5); it serves the stage-2 metadata mode of [CMD-15.5](application.md#cmd-15-5), which writes no chunks or vectors ([ING-20](ingestion.md#ing-20)).
- <a id="adp-09-3"></a>**ADP-09.3 Keyword backend switch.** With `DOCSTORE_SEARCH__KEYWORD_BACKEND=tsvector` ([ADR-03.2](decisions.md#adr-03-2) fallback), the BM25 index is replaced by a generated `tsv tsvector` column with a GIN index, and the read model switches its keyword leg. The writer is unchanged.
- <a id="adp-09-4"></a>**ADP-09.4 Bulk-load mode.** For an initial bulk load, [ADR-12.3](decisions.md#adr-12-3) builds HNSW after loading. The operator drops the HNSW index before a first full run and recreates it afterwards with `CREATE INDEX CONCURRENTLY` (script `deploy/sql/hnsw-rebuild.sql`), with `maintenance_work_mem` raised per [DEP-02.4](overview.md#dep-02-4). Search runs keyword-only, flagged `degraded`, while the index is absent.
- <a id="adp-09-5"></a>**ADP-09.5 Document-level key table.** `document_enrichment_keys` has one row per document, keyed by `document_id`, with the document-level keys of [DM-05.3](domain.md#dm-05-3) and a B-tree or GIN index per filterable column. Writes are `INSERT … ON CONFLICT (document_id) DO UPDATE`. The table is not part of the HNSW or BM25 index, so a stage-2 write is one narrow row per document: no `chunks` tuple, no HNSW graph insert and no BM25 entry. The hybrid statement of [ADP-10.1](#adp-10-1) joins it inside each leg when a document-level filter is present ([ADR-17.2](decisions.md#adr-17-2)).

Refs: [PORT-10](#port-10), [ADR-03](decisions.md#adr-03), [ADR-12](decisions.md#adr-12)

<a id="adp-10"></a>
### ADP-10 Postgres read model

Implements [PORT-11](#port-11), module [MOD-06.2](overview.md#mod-06-2). It connects as `docstore_reader` with the row-level-security settings of [XC-01.6](overview.md#xc-01-6).

- <a id="adp-10-1"></a>**ADP-10.1 Hybrid statement.** One statement per search, with bound parameters:

  ```sql
  WITH acl AS (SELECT :allowed::text[] AS allowed, :walled::text[] AS walled),
  kw AS (
    SELECT c.id, c.document_id, row_number() OVER (ORDER BY paradedb.score(c.id) DESC) AS r
    FROM chunks c, acl
    WHERE c.text @@@ :q
      AND c.access_keys && acl.allowed AND NOT c.access_keys && acl.walled
      AND <metadata filters>
    ORDER BY paradedb.score(c.id) DESC LIMIT :depth),
  vec AS (
    SELECT c.id, c.document_id, row_number() OVER (ORDER BY c.embedding <=> :qvec) AS r
    FROM chunks c, acl
    WHERE c.embedding_model = :model
      AND c.access_keys && acl.allowed AND NOT c.access_keys && acl.walled
      AND <metadata filters>
    ORDER BY c.embedding <=> :qvec LIMIT :depth),
  fused AS (
    SELECT id, document_id, SUM(1.0 / (:k + r)) AS rrf
    FROM (SELECT * FROM kw UNION ALL SELECT * FROM vec) u GROUP BY id, document_id)
  SELECT document_id, MAX(rrf) AS score, (ARRAY_AGG(id ORDER BY rrf DESC))[1:3] AS best_chunks
  FROM fused GROUP BY document_id ORDER BY score DESC LIMIT :limit OFFSET :offset;
  ```

  `<metadata filters>` on document-level keys ([DM-05.3](domain.md#dm-05-3)) are an `EXISTS` join to `document_enrichment_keys` on `document_id` ([ADP-09.5](#adp-09-5)); chunk-level keys are filtered on the chunk row. The permission predicate of [XC-01.3](overview.md#xc-01-3) is inside each leg, before `ORDER BY … LIMIT`, so ranking, fusion and depth only ever see visible chunks. `:depth` and `:k` are the `candidates_per_leg` and `rrf_k` inputs of [QRY-01.1](application.md#qry-01-1). The duplicate exclusion and optional near-duplicate collapse of [QRY-01.3](application.md#qry-01-3) are extra predicates in the final `SELECT`. Snippets come from `paradedb.snippet` on the best chunk, or `ts_headline` in tsvector mode.
- <a id="adp-10-2"></a>**ADP-10.2 Vector leg settings.** Each search transaction runs `SET LOCAL hnsw.ef_search` and `SET LOCAL hnsw.iterative_scan = relaxed_order`, so selective permission filters do not under-fill the top-k ([XC-01.4](overview.md#xc-01-4)). Values are tuned per [ADR-12.2](decisions.md#adr-12-2).
- <a id="adp-10-3"></a>**ADP-10.3 Modes.** `keyword` and `vector` modes execute only their leg and rank by it. A degraded hybrid search (no query vector, [PORT-11.4](#port-11-4)) runs the keyword leg and sets `degraded=true`.
- <a id="adp-10-4"></a>**ADP-10.4 Facets and totals.** Computed over the matched, visible document set with the same `acl` CTE and filters: `COUNT(DISTINCT document_id)` grouped per facet field. `total_estimate` counts at most 10,001 documents, so totals above 10,000 are reported as `10000+`.
- <a id="adp-10-5"></a>**ADP-10.5 Views.** Non-search reads use these SQL views (files per [MOD-08.3](overview.md#mod-08-3)): `v_document_detail` ([QRY-02](application.md#qry-02), [QRY-03](application.md#qry-03) with the blob store), `v_thread_messages` ([QRY-04](application.md#qry-04)), `v_client_list` ([QRY-05](application.md#qry-05)), `v_matter_list` ([QRY-06](application.md#qry-06)), `v_matter_overview` ([QRY-07](application.md#qry-07)), `v_review_queue` ([QRY-09](application.md#qry-09)), `v_run_progress` ([QRY-10](application.md#qry-10)), `v_run_failures` ([QRY-11](application.md#qry-11)), `v_taxonomy` ([QRY-12](application.md#qry-12)), `v_audit` ([QRY-14](application.md#qry-14)), `v_person_list` ([QRY-15](application.md#qry-15)). Views that expose documents, matters, clients or persons are queried with the same `acl` predicate on their access-key column.
- <a id="adp-10-6"></a>**ADP-10.6 Similar documents.** `similar` implements [QRY-13.3](application.md#qry-13-3) as one statement: a `src` CTE computing `l2_normalize(avg(embedding))` over the source's active chunks, followed by the vector leg of [ADP-10.1](#adp-10-1) with the same predicate and the source's duplicate set excluded.
- <a id="adp-10-7"></a>**ADP-10.7 Progress.** `run_progress` reads `v_run_progress` (counters of [ING-16](ingestion.md#ing-16)). Live workflow state is merged by [QRY-10](application.md#qry-10) from [PORT-18](#port-18), not by this adapter.

Refs: [PORT-11](#port-11), [XC-01](overview.md#xc-01), [ADR-03](decisions.md#adr-03), [ADR-09](decisions.md#adr-09)

<a id="adp-11"></a>
### ADP-11 Voyage embedding client

**Retired:** there is no external embedding provider; the local endpoint client is the default. Replaced by [ADP-12](#adp-12).

- <a id="adp-11-1"></a>**ADP-11.1 Call.** **Retired:** replaced by [ADP-12.1](#adp-12-1).
- <a id="adp-11-2"></a>**ADP-11.2 Limits.** **Retired:** batch limits are now configured endpoint limits. Replaced by [ADP-12.1](#adp-12-1).
- <a id="adp-11-3"></a>**ADP-11.3 Local token bucket.** **Retired:** there is no provider quota to share. Replaced by [ADP-12.5](#adp-12-5).
- <a id="adp-11-4"></a>**ADP-11.4 Status mapping.** **Retired:** replaced by [ADP-12.4](#adp-12-4).
- <a id="adp-11-5"></a>**ADP-11.5 Query path.** **Retired:** replaced by [ADP-12.6](#adp-12-6).
- <a id="adp-11-6"></a>**ADP-11.6 Model reference.** **Retired:** replaced by [ADP-12.8](#adp-12-8).
- <a id="adp-11-7"></a>**ADP-11.7 Logging.** **Retired:** replaced by [ADP-12.8](#adp-12-8).

Refs: [ADP-12](#adp-12), [ADR-06](decisions.md#adr-06), [ADR-16](decisions.md#adr-16)

<a id="adp-12"></a>
### ADP-12 Local embedding client (OpenAI-compatible)

Default [PORT-12](#port-12) implementation, module [MOD-06.3](overview.md#mod-06-3). It talks to the local `embed` endpoint ([DEP-09](overview.md#dep-09)) and depends only on base URL, model name and parameters from configuration ([XC-04](overview.md#xc-04)).

- <a id="adp-12-1"></a>**ADP-12.1 Call.** `httpx` `POST {embedding.BASE_URL}/embeddings` (default `http://embed:8000/v1`, internal only, [DEP-09](overview.md#dep-09)) with `{"model": embedding.MODEL, "input": [...], "encoding_format": "float"}`. Queries are prefixed with `embedding.QUERY_INSTRUCTION`; documents are sent unprefixed ([PORT-12.1](#port-12-1)). The batch `limits` are `embedding.MAX_BATCH_TEXTS` (64) and `embedding.MAX_BATCH_TOKENS` (32,768) from settings, not constants in code. The read timeout is `embedding.TIMEOUT_S`.
- <a id="adp-12-2"></a>**ADP-12.2 Other endpoints.** llama.cpp server with `--embedding` and Ollama serve the same path and are selected by configuration ([DEP-09.4](overview.md#dep-09-4)). A runtime change with identical weight files keeps the `embedding_model` stamp but must pass the probe check of [ADP-12.7](#adp-12-7). Different weights mean a new stamp and a full re-embed ([ADR-12.3](decisions.md#adr-12-3), [ING-17.5](ingestion.md#ing-17-5)).
- <a id="adp-12-3"></a>**ADP-12.3 Matryoshka truncation.** The adapter takes the first `embedding.DIMENSION` (1024) components of each returned vector and L2-normalises them, so the `halfvec(1024)` schema stays ([ADR-16.2](decisions.md#adr-16-2)). It never relies on server-side `dimensions` support. For native-1024 models (BGE-M3) the truncation is a no-op. A returned vector shorter than `DIMENSION` raises `PermanentError("dimension_mismatch")` ([PORT-12.4](#port-12-4)).
- <a id="adp-12-4"></a>**ADP-12.4 Status mapping.** Connection refused, 429 and 503 → `BackpressureError` (`Retry-After`, or 5 s). Read timeout → `TransientError`. Other 5xx → `TransientError`. 400 and 413 → `PermanentError("embedding_rejected")`. There is no authentication mapping, because the internal endpoint takes no API key ([XC-02.5](overview.md#xc-02-5)).
- <a id="adp-12-5"></a>**ADP-12.5 Concurrency.** Ingest calls are bounded by the `embed` queue cap ([WF-04.8](ingestion.md#wf-04-8)). Query calls in api and mcp, which do not run through DBOS queues, are bounded per process by a semaphore of `embedding.QUERY_CONCURRENCY`. Query calls bypass the ingest queue, which gives them admission priority under the GPU sharing policy ([WF-04.9](ingestion.md#wf-04-9)); inside the `embed` server they share FIFO scheduling with ingest batches. There is no token bucket.
- <a id="adp-12-6"></a>**ADP-12.6 Query path.** `embed_query` has a 2 s deadline and one retry, so [QRY-01](application.md#qry-01) can fall back to keyword in time. A per-process LRU cache (1,024 entries, keyed by model and normalised text) avoids re-embedding repeated queries.
- <a id="adp-12-7"></a>**ADP-12.7 Readiness and warm-up.** `GET /v1/models` must list `embedding.MODEL`, the digest-bearing served name ([DEP-10.1](overview.md#dep-10-1)). Warm-up embeds the probe set of [DEP-10.5](overview.md#dep-10-5) and requires cosine ≥ 0.999 to the reference vectors after truncation. Until ready, calls raise `BackpressureError`, so lanes wait instead of failing ([SC-16.5](../success-criteria.md#sc-16-5)).
- <a id="adp-12-8"></a>**ADP-12.8 Model reference and logging.** The adapter reports the `embedding_model` stamp of [XC-06.1](overview.md#xc-06-1), which [CMD-14](application.md#cmd-14) stamps on chunks. It logs the per-call metrics of [XC-03.3](overview.md#xc-03-3), never request text ([XC-02.7](overview.md#xc-02-7)).
- <a id="adp-12-9"></a>**ADP-12.9 Status.** `status()` ([PORT-12.7](#port-12-7)) reads `GET /v1/models` and `GET /metrics` of the `embed` endpoint and the cached readiness and warm-up result of [ADP-12.7](#adp-12-7), and compares the served name with the `embedding` entry of `models.lock`. It sends no embedding request.

Refs: [PORT-12](#port-12), [DEP-09](overview.md#dep-09), [DEP-10](overview.md#dep-10), [WF-04](ingestion.md#wf-04), [ADR-16](decisions.md#adr-16)

<a id="adp-13"></a>
### ADP-13 Local LLM client (OpenAI-compatible)

Implements [PORT-13](#port-13), module [MOD-06.4](overview.md#mod-06-4). Model, prompt and runtime parameters are owned by [LLM-01](ingestion.md#llm-01).

- <a id="adp-13-1"></a>**ADP-13.1 Call.** `httpx` `POST {llm.BASE_URL}/chat/completions` (default `http://llm:8000/v1`, vLLM, internal only, [DEP-07](overview.md#dep-07)) with `model` = `llm.MODEL`, `messages` (system, user), `temperature`, `seed`, `max_tokens` and `response_format = {"type": "json_schema", "json_schema": {"name": schema_name, "schema": json_schema, "strict": true}}`. The adapter merges `llm.EXTRA_BODY` (model- and runtime-specific parameters, for example `reasoning_effort`) into the request. It discards any reasoning content and never stores or logs it. The same request works against llama.cpp server and Ollama ([DEP-07.6](overview.md#dep-07-6)).
- <a id="adp-13-2"></a>**ADP-13.2 Concurrency semaphore.** A process-wide `threading.BoundedSemaphore(DOCSTORE_QUEUES__LLM_CONCURRENCY)` guards calls, as defence in depth behind the `enrich` queue cap ([WF-04.3](ingestion.md#wf-04-3)) and the server's `--max-num-seqs` ([DEP-07.5](overview.md#dep-07-5)).
- <a id="adp-13-3"></a>**ADP-13.3 Timeouts and errors.** The read timeout is `llm.TIMEOUT_S` (default 180 s). Connection refused, 429 and 503 → `BackpressureError` (`Retry-After`, or 30 s, [ING-15.4](ingestion.md#ing-15-4)). Timeouts and other 5xx → `TransientError`. 400 (for example, a context overflow) → `PermanentError("llm_rejected")`. Invalid output → `InvalidLlmOutput` ([PORT-13.1](#port-13-1)).
- <a id="adp-13-4"></a>**ADP-13.4 Readiness and warm-up.** Readiness calls `GET /v1/models` and requires `llm.MODEL`, whose served name carries the weights digest ([LLM-01.3](ingestion.md#llm-01-3)); the digest is recorded for `model_digest`. Warm-up sends one fixed synthetic structured request and requires a schema-valid answer. Until ready, calls raise `BackpressureError`, so the `enrich` lane waits instead of failing every step ([SC-16.5](../success-criteria.md#sc-16-5)). This is the only mechanism that gates the `enrich` lane on the `llm` service: workers keep listening on every queue ([WF-04.1](ingestion.md#wf-04-1)), and stage 1 and search are unaffected ([DEP-01.6](overview.md#dep-01-6)).
- <a id="adp-13-5"></a>**ADP-13.5 Status.** `status()` ([PORT-13.4](#port-13-4)) reads `GET /v1/models` and `GET /metrics` of the `llm` endpoint and the cached result of [ADP-13.4](#adp-13-4), and compares the served name with the `llm` entry of `models.lock`. It sends no completion request.

Refs: [PORT-13](#port-13), [LLM-01](ingestion.md#llm-01), [DEP-07](overview.md#dep-07), [ADR-15](decisions.md#adr-15)

<a id="adp-14"></a>
### ADP-14 Tika extractor

Implements format detection and non-e-mail extraction for [PORT-14](#port-14), module [MOD-06.5](overview.md#mod-06-5), against the tika service ([DEP-06](overview.md#dep-06)).

- <a id="adp-14-1"></a>**ADP-14.1 Detection.** `PUT /detect/stream` with the `Content-Disposition` filename hint returns the MIME type. The adapter maps it to a format class of [TAX-06](domain.md#tax-06) through a MIME-type table in the adapter.
- <a id="adp-14-2"></a>**ADP-14.2 Extraction.** `PUT /rmeta/text` returns a JSON list: the container first, then one entry per embedded resource (`X-TIKA:embedded_resource_path`). The container's `X-TIKA:content` becomes `text`. Metadata keys are normalised: `dc:title` → `title`, `dc:creator` → `author`, `dcterms:created` → `created`, `dcterms:modified` → `modified`, `xmpTPg:NPages` → `page_count`, and `pdf:charsPerPage` → `chars_per_page`.
- <a id="adp-14-3"></a>**ADP-14.3 Children.** When embedded resources exist (Office attachments, ZIP members, PDF portfolios), `PUT /unpack/all` returns a ZIP of the embedded files, which is unpacked into `workdir` as children ([PORT-14.3](#port-14-3)). Inline images below 50 KB are not emitted as children.
- <a id="adp-14-4"></a>**ADP-14.4 Limits.** Request header `X-Tika-Timeout-Millis` defaults to 120,000. Inputs over the configured size limit (default 512 MB) are rejected before upload with `PermanentError("too_large")`. Tika OCR is disabled server-side ([DEP-06.2](overview.md#dep-06-2)).
- <a id="adp-14-5"></a>**ADP-14.5 Status mapping.** 422 (encrypted or corrupt) → `PermanentError` with Tika's reason mapped to `encrypted` or `corrupt`. 415 → `unsupported_format`. 5xx, 503 during a child restart, and timeouts → `TransientError`.
- <a id="adp-14-6"></a>**ADP-14.6 Version.** `parser_version = "tika-<server version>/docstore-extract-<n>"`, where the server version is read from `GET /version` at startup ([XC-06.1](overview.md#xc-06-1)).

Refs: [PORT-14](#port-14), [DEP-06](overview.md#dep-06), [ADR-07](decisions.md#adr-07)

<a id="adp-15"></a>
### ADP-15 E-mail extractor (EML/MSG)

Implements [PORT-14](#port-14) for e-mail formats, routed by [PORT-14.1](#port-14-1), and runs in the worker image.

- <a id="adp-15-1"></a>**ADP-15.1 EML.** `email.parser.BytesParser(policy=email.policy.default)`. The adapter reads the headers `Message-ID`, `In-Reply-To`, `References`, `From`, `To`, `Cc`, `Bcc`, `Date` and `Subject` into `EmailHeaders`, with RFC 2047 decoding and addresses lower-cased. The body is the `text/plain` part if present. Otherwise the `text/html` part is converted to text through [ADP-14](#adp-14), so there is one HTML conversion path.
- <a id="adp-15-2"></a>**ADP-15.2 MSG.** `extract_msg.openMsg(path)` gives the same headers, taken from the transport headers when present and otherwise from MAPI properties. The body comes from plain text, else HTML, else RTF de-encapsulated to HTML. Embedded `.msg` attachments become children of type `application/vnd.ms-outlook`.
- <a id="adp-15-3"></a>**ADP-15.3 Attachments.** Each attachment is written to `workdir` with its declared filename (sanitised), and its content ID is kept for inline parts. Signed and encrypted S/MIME parts: a signed message is unwrapped, and an encrypted one is extracted as headers-only with warning `smime_encrypted`.
- <a id="adp-15-4"></a>**ADP-15.4 Version.** `parser_version = "pyemail/extract-msg-<ver>/docstore-mail-<n>"`.

Refs: [PORT-14](#port-14), [ADR-07](decisions.md#adr-07)

<a id="adp-16"></a>
### ADP-16 OCRmyPDF / Tesseract

Implements [PORT-15](#port-15) inside the worker image ([DEP-05](overview.md#dep-05)), module [MOD-06.5](overview.md#mod-06-5).

- <a id="adp-16-1"></a>**ADP-16.1 Invocation.** The adapter runs the `ocrmypdf` CLI as a subprocess, not in process, so that a crash or hang kills only the child, which [XC-05.1](overview.md#xc-05-1) classifies as `poison` after repeats. Arguments: `--skip-text --rotate-pages --deskew --output-type pdf --sidecar <txt> --language <eng+pol…> --jobs 1 --tesseract-timeout 180 --optimize 0`. Single-image inputs are wrapped into a PDF with `img2pdf` first, and multi-page TIFFs are converted page by page.
- <a id="adp-16-2"></a>**ADP-16.2 Confidence.** Page `mean_confidence` is the mean Tesseract word confidence (hOCR `x_wconf`) captured during the run. It is `None` when the renderer does not expose it, and the caller must handle `None` ([PORT-15](#port-15)).
- <a id="adp-16-3"></a>**ADP-16.3 Timeouts and exit codes.** The overall subprocess timeout is `DOCSTORE_OCR__TIMEOUT_S` (default 900 s), plus 120 s per page above five pages. OCRmyPDF exit code 6 (already has text) is success with no new text. Codes 2, 4 and 8 (bad input, invalid output, encrypted) → `PermanentError`. Timeout or signal → `TransientError`, counted towards poison by the step.
- <a id="adp-16-4"></a>**ADP-16.4 Version.** `ocr_version = "ocrmypdf-<ver>/tesseract-<ver>/<langs>"`, read once at startup with `--version`.

Refs: [PORT-15](#port-15), [DEP-05](overview.md#dep-05), [ADR-07](decisions.md#adr-07)

<a id="adp-17"></a>
### ADP-17 Filesystem connector

Implements [PORT-16](#port-16) for local or mounted folders (mounted read-only into the worker), module [MOD-06.6](overview.md#mod-06-6).

- <a id="adp-17-1"></a>**ADP-17.1 Walk and cursor.** A depth-first `os.scandir` walk with entries sorted by NFC-normalised relative path gives a total order. The cursor is `{"after": "<relpath>"}`, and a page is the next `limit` files after it. Symlinks are not followed.
- <a id="adp-17-2"></a>**ADP-17.2 Key and fingerprint.** `key = "fs:" + relpath`. `fingerprint = f"{size}:{mtime_ns}"`. `source_metadata` holds the path segments (`segment_1` …), which [DM-15](domain.md#dm-15) path rules match.
- <a id="adp-17-3"></a>**ADP-17.3 Filters.** gitignore-style include and exclude globs (`pathspec`) come from the source config. Default excludes are `~$*`, `*.tmp`, `Thumbs.db`, `.DS_Store`, `desktop.ini` and `*.pst`. A file that matches an exclude glob is reported with `excluded_by` set to that glob rather than dropped ([DM-02.6](domain.md#dm-02-6)). PST files are registered as PST sources ([ADP-18](#adp-18)) rather than walked as files.
- <a id="adp-17-4"></a>**ADP-17.4 Deletions.** After a walk completes, items of the source not seen in that run are reported as `deleted=True`, by comparing the last-seen run ID in [PORT-07](#port-07).

Refs: [PORT-16](#port-16)

<a id="adp-18"></a>
### ADP-18 PST archive connector

Implements [PORT-16](#port-16) for Outlook PST/OST archive files, module [MOD-06.6](overview.md#mod-06-6).

- <a id="adp-18-1"></a>**ADP-18.1 Enumeration.** `pypff.file().open(path)` walks `root_folder` depth-first in sub-folder index order. The cursor is `{"folder_path": [i, j, …], "message_index": n}`. Each message is one item. The source config names the PST owner (mailbox owner), which goes into `source_metadata` for rules.
- <a id="adp-18-2"></a>**ADP-18.2 Key and fingerprint.** `key = "pst:<PST path relative to the source root>:<folder path>:<message identifier>"`. The libpff message identifier is stable within an unchanged file. The fingerprint is the PST file's SHA-256, so an unchanged archive is skipped entirely on re-discovery.
- <a id="adp-18-3"></a>**ADP-18.3 Fetch as EML.** `fetch` rebuilds the message as RFC 822 `.eml` with `email.message.EmailMessage`, using the transport headers when present and otherwise headers synthesised from MAPI properties (sender, recipients, subject, client submit time). It adds the plain or HTML body and the attachments (`get_attachment(i).read_buffer`). Synthesised headers are marked `X-Docstore-Headers-Synthesised: true`. Downstream extraction is then [ADP-15](#adp-15) only.
- <a id="adp-18-4"></a>**ADP-18.4 Local copy and corruption.** The PST is read from a local volume, not over SMB. A message that libpff cannot read raises `PermanentError("corrupt")` for that item only, and enumeration continues.

Refs: [PORT-16](#port-16), [ADR-07](decisions.md#adr-07)

<a id="adp-19"></a>
### ADP-19 SharePoint connector

Implements [PORT-16](#port-16) for SharePoint document libraries in two modes with identical keys, module [MOD-06.6](overview.md#mod-06-6).

- <a id="adp-19-1"></a>**ADP-19.1 Stable key in both modes.** `key = "sp:<site id>:<drive id>:<item id>"`. A source that starts from an export can therefore switch to live mode without re-creating documents.
- <a id="adp-19-2"></a>**ADP-19.2 Export mode (PoC default).** The adapter reads a folder export produced by `deploy/scripts/export-sharepoint.ps1` (PnP.PowerShell). The export has the layout `<root>/<site>/<library>/<server-relative path>` plus `manifest.jsonl` with one line per file: `site_id`, `site_url`, `drive_id`, `item_id`, `path`, `created`, `modified`, `author`, `editor`, `etag`, and the library's custom columns under `fields`. Paging and cursor work as in [ADP-17.1](#adp-17-1), over manifest order, and the fingerprint is the `etag`. It needs no egress ([DEP-01.5](overview.md#dep-01-5)) and is the PoC path ([F-01.4](../features.md#f-01-4)).
- <a id="adp-19-3"></a>**ADP-19.3 Live mode (later).** Live mode requires Graph egress, which the zero-egress posture excludes ([DEP-01.5](overview.md#dep-01-5), [SCOPE-04.7](../README.md#scope-04-7)); it is described here so that the keys of [ADP-19.1](#adp-19-1) stay stable when it is enabled. `msgraph-sdk` with app-only credentials (`Sites.Selected`, granted per site). Discovery uses `GET /drives/{drive-id}/root/delta` per library. The cursor holds the `@odata.deltaLink` per drive, or the `@odata.nextLink` mid-page. The fingerprint is `cTag`. The `deleted` facet produces tombstones. Library columns come from `listItem/fields` with `$expand`. Graph 429 and 503 map to `BackpressureError` with `Retry-After`. The async SDK is bridged per [PORT-20.1](#port-20-1).
- <a id="adp-19-4"></a>**ADP-19.4 Metadata.** In both modes, `source_metadata` carries `site_url`, `library`, `path` and the custom columns, which [DM-15](domain.md#dm-15) site and column rules match. Document-level SharePoint permissions are not imported in the PoC ([F-16](../features.md#f-16)).

Refs: [PORT-16](#port-16)

<a id="adp-20"></a>
### ADP-20 Filesystem blob store

Implements [PORT-17](#port-17) on the `blobs` volume ([DEP-08](overview.md#dep-08)), module [MOD-06.7](overview.md#mod-06-7).

- <a id="adp-20-1"></a>**ADP-20.1 Layout.** `<root>/sha256/ab/cd/<full hex>`, where `ab` and `cd` are the first two byte pairs. This gives 65,536 leaf directories, which keeps directory sizes small at tens of millions of blobs. Extracted text is stored the same way (`put_text`, UTF-8).
- <a id="adp-20-2"></a>**ADP-20.2 Atomic put.** The adapter streams into `<root>/tmp/<uuid>` while hashing, then runs `fsync`, `os.replace` to the final path and an `fsync` of the directory. If the final path exists, it deletes the temporary file and returns ([PORT-17.2](#port-17-2)).
- <a id="adp-20-3"></a>**ADP-20.3 Read-only originals.** Final files are `chmod 0444`. `local_path` returns the final path, which the worker passes to Tika and OCRmyPDF without copying.
- <a id="adp-20-4"></a>**ADP-20.4 Later swap.** A MinIO/S3 adapter keeps the same keys (object name = hex digest) and returns `None` from `local_path` ([ADR-11](decisions.md#adr-11)).

Refs: [PORT-17](#port-17), [ADR-11](decisions.md#adr-11)

<a id="adp-21"></a>
### ADP-21 In-process event publisher and audit writer

Implements [PORT-19](#port-19), module [MOD-06.9](overview.md#mod-06-9).

- <a id="adp-21-1"></a>**ADP-21.1 Dispatch loop.** For each event, the adapter calls the subscribers in registration order. Each subscriber call is wrapped so that an exception is logged with `event_id` and does not stop later subscribers ([ARCH-05.3](overview.md#arch-05-3)).
- <a id="adp-21-2"></a>**ADP-21.2 Audit subscriber.** It writes one `audit_log` row per event of a `worker` command ([XC-08.2](overview.md#xc-08-2)), batching all events of one commit into one `INSERT`. Events of `human` or `agent` commands are skipped, because their entry was written in the command transaction ([PORT-04.5](#port-04-5)).
- <a id="adp-21-3"></a>**ADP-21.3 Access entries.** `record_access` inserts one row as `docstore_reader`, which has `INSERT` on `audit_log` only ([XC-08.4](overview.md#xc-08-4)), in autocommit mode, and propagates any failure ([PORT-19.3](#port-19-3)).

Refs: [PORT-19](#port-19), [XC-08](overview.md#xc-08), [ADR-13](decisions.md#adr-13)

---

## Test doubles

<a id="adp-22"></a>
### ADP-22 In-memory test doubles

Module [MOD-06.11](overview.md#mod-06-11), bound by the `test` profile ([ARCH-07.2](overview.md#arch-07-2)). Each fake passes the port's contract suite ([PORT-20.6](#port-20-6)).

| ID | Fake | Port | Behaviour worth knowing |
|---|---|---|---|
| <a id="adp-22-1"></a>ADP-22.1 | `InMemoryUnitOfWork` and in-memory repositories | [PORT-04](#port-04) to [PORT-09](#port-09) | Dict-backed, with copy-on-commit, version checks raising `ConcurrencyConflict`, and unique-key checks |
| <a id="adp-22-2"></a>ADP-22.2 | `InMemoryIndex`, `InMemoryReadModel` | [PORT-10](#port-10), [PORT-11](#port-11) | Naive term-overlap keyword leg, cosine vector leg, and RRF. It applies the same `AccessScope` predicate, so application-level wall tests run without Postgres |
| <a id="adp-22-3"></a>ADP-22.3 | `HashEmbedder` | [PORT-12](#port-12) | Deterministic 1024-d vectors from a text hash. It can be scripted to raise `BackpressureError` on the n-th call |
| <a id="adp-22-4"></a>ADP-22.4 | `ScriptedLlm` | [PORT-13](#port-13) | Returns queued responses or invalid JSON on demand, and records requests for prompt-contract assertions |
| <a id="adp-22-5"></a>ADP-22.5 | `FixtureExtractor`, `FixtureOcr` | [PORT-14](#port-14), [PORT-15](#port-15) | Return canned `Extraction` and `OcrResult` values per fixture filename |
| <a id="adp-22-6"></a>ADP-22.6 | `ListConnector` | [PORT-16](#port-16) | Pages over an in-memory list with an index cursor, and can be mutated between pages to test fingerprint changes |
| <a id="adp-22-7"></a>ADP-22.7 | `MemoryBlobStore` | [PORT-17](#port-17) | Dict keyed by SHA-256 |
| <a id="adp-22-8"></a>ADP-22.8 | `ImmediateWorkflowEngine` | [PORT-18](#port-18) | Records started specs and dedupes by ID. Tests drive the steps explicitly |
| <a id="adp-22-9"></a>ADP-22.9 | `RecordingPublisher`, `FakeAuthenticator` | [PORT-19](#port-19), [PORT-03](#port-03) | Record events and access entries; map fixed tokens to principals |
| <a id="adp-22-10"></a>ADP-22.10 | `FaultInjectingEndpoint` | model endpoints, used by [ADP-12](#adp-12) and [ADP-13](#adp-13) | An HTTP shim in the compose `test` profile, in front of `llm` and `embed`, that returns 503, 429 or connection resets at a configured rate and window. Used in chaos and induced tests ([SC-09.1](../success-criteria.md#sc-09-1)) |

Refs: [PORT-04](#port-04), [PORT-05](#port-05), [PORT-11](#port-11), [PORT-12](#port-12), [PORT-13](#port-13), [PORT-16](#port-16), [PORT-18](#port-18), [XC-07](overview.md#xc-07), [SC-09](../success-criteria.md#sc-09)
