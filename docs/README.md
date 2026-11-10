# Law-firm document store PoC: documentation

Architecture and acceptance documentation for a local proof of concept of an AI-friendly document store. The store ingests a law firm's e-mail, Office, PDF, SharePoint and file-share content, enriches it with a local LLM, assigns it to clients, matters, handlers and case types, and serves permission-aware hybrid search to people and to AI agents over MCP.

This file owns the documentation conventions (CONV), the glossary (GL) and the PoC scope (SCOPE). Every other fact lives in the file that owns its ID; see [CONV-05](#conv-05).

## How to read

<a id="conv-06"></a>
### CONV-06 Reading order and priority labels

Read the files in this order. Each step names the question the file answers.

- <a id="conv-06-1"></a>**CONV-06.1** [README.md](README.md): conventions, vocabulary and the PoC boundary (this file).
- <a id="conv-06-2"></a>**CONV-06.2** [success-criteria.md](success-criteria.md): what the PoC must prove and how each criterion is measured. Start at [SC-01](success-criteria.md#sc-01).
- <a id="conv-06-3"></a>**CONV-06.3** [features.md](features.md): what the system does, grouped by area and prioritised. Start at [F-01](features.md#f-01).
- <a id="conv-06-4"></a>**CONV-06.4** [architecture/overview.md](architecture/overview.md): the hexagon map, layers, deployment, module layout and cross-cutting concerns. Start at [ARCH-01](architecture/overview.md#arch-01).
- <a id="conv-06-5"></a>**CONV-06.5** [architecture/domain.md](architecture/domain.md): entities, aggregates, invariants, domain events and taxonomies.
- <a id="conv-06-6"></a>**CONV-06.6** [architecture/application.md](architecture/application.md): the commands and queries that form the application API.
- <a id="conv-06-7"></a>**CONV-06.7** [architecture/ports-adapters.md](architecture/ports-adapters.md): the ports and the adapters that implement them, with PoC technology.
- <a id="conv-06-8"></a>**CONV-06.8** [architecture/ingestion.md](architecture/ingestion.md): the durable ingestion workflow, pipeline steps and the LLM enrichment contract.
- <a id="conv-06-9"></a>**CONV-06.9** [architecture/mcp.md](architecture/mcp.md): the MCP server, its tools and permission handling.
- <a id="conv-06-10"></a>**CONV-06.10** [architecture/decisions.md](architecture/decisions.md): why each significant choice was made and what was rejected.
- <a id="conv-06-11"></a>**CONV-06.11** Shortcut for reviewers of a single concern: start at the success criterion, follow its links to features, then to the commands, queries, ports and steps that realise them.
- <a id="conv-06-12"></a>**CONV-06.12** Priority label **must**: required for the PoC to pass its success criteria; the PoC is not done without it.
- <a id="conv-06-13"></a>**CONV-06.13** Priority label **should**: planned for the PoC and built if time allows; no success criterion fails without it.
- <a id="conv-06-14"></a>**CONV-06.14** Priority label **later**: designed for (a port, field or extension point exists) but not built in the PoC.

Refs: [SC-01](success-criteria.md#sc-01), [F-01](features.md#f-01), [ARCH-01](architecture/overview.md#arch-01)

## Conventions

<a id="conv-01"></a>
### CONV-01 ID scheme

- <a id="conv-01-1"></a>**CONV-01.1** A top-level item has the ID `PREFIX-NN`: an upper-case prefix from [CONV-05](#conv-05), a hyphen and a two-digit number. Numbering starts at 01 for each prefix. A prefix that passes 99 continues with three digits.
- <a id="conv-01-2"></a>**CONV-01.2** A sub-item has the ID `PREFIX-NN.M`, where M starts at 1 within its parent and is not zero-padded. Sub-items are not nested further.
- <a id="conv-01-3"></a>**CONV-01.3** IDs are never reused or renumbered. A dropped item keeps its ID and heading, is marked **Retired** with a one-line reason, and its number stays taken.
- <a id="conv-01-4"></a>**CONV-01.4** A new item takes the next free number of its prefix, and only the owning file ([CONV-05](#conv-05)) may add it.
- <a id="conv-01-5"></a>**CONV-01.5** "Item" covers every criterion, feature, component, entity, command, query, port, adapter, pipeline step, MCP tool, decision and glossary term. It also covers every meaningful sub-point: an acceptance check, invariant, rule or field group.

Refs: [CONV-05](#conv-05)

<a id="conv-02"></a>
### CONV-02 Anchors

- <a id="conv-02-1"></a>**CONV-02.1** The anchor of an item is its ID in lower case with dots replaced by hyphens.
- <a id="conv-02-2"></a>**CONV-02.2** Heading item: the anchor sits on its own line directly above the heading, which starts with the ID.

  ```markdown
  <a id="xx-07"></a>
  ### XX-07 Title
  ```
- <a id="conv-02-3"></a>**CONV-02.3** List sub-item: the anchor opens the list entry, followed by the bold ID.

  ```markdown
  - <a id="xx-07-2"></a>**XX-07.2** text
  ```
- <a id="conv-02-4"></a>**CONV-02.4** Table row: the first cell holds the anchor followed by the ID.

  ```markdown
  | <a id="xx-07"></a>XX-07 | Title | ... |
  ```
- <a id="conv-02-5"></a>**CONV-02.5** Generated heading slugs are never used as link targets, because they change when a title changes. Only explicit anchors are used.

Refs: [CONV-01](#conv-01)

<a id="conv-03"></a>
### CONV-03 Linking and Refs lines

- <a id="conv-03-1"></a>**CONV-03.1** Every mention of an ID is a relative Markdown link to its anchor, with the ID alone as link text: `[XX-07](path/to/file.md#xx-07)`, or `[XX-07](#xx-07)` within the same file. Paths are relative to the linking file.
- <a id="conv-03-2"></a>**CONV-03.2** Bare IDs are not allowed in prose or tables. The only exceptions are an item's own heading or bold label, placeholder IDs (`XX-…`) inside code examples, and Mermaid diagram labels, which cannot hold links: an ID in a diagram label is allowed when the item that owns the diagram links it, in a legend item under the diagram (as [ARCH-01.2](architecture/overview.md#arch-01-2) does) or in its text or Refs line, or when the node has a Mermaid `click` link.
- <a id="conv-03-3"></a>**CONV-03.3** An item lists its outgoing references in one line starting with `Refs:`, as comma-separated links, placed at the end of the item. Incoming references are not listed: they are found by search.
- <a id="conv-03-4"></a>**CONV-03.4** A link may carry at most a short phrase of context (`rate limited by [XX-04](#xx-04)`). Anything longer means a fact is being restated (see [CONV-04](#conv-04)).
- <a id="conv-03-5"></a>**CONV-03.5** External sources (vendor docs, specs) are linked by URL only in the item that owns the fact they support.

Refs: [CONV-02](#conv-02), [CONV-04](#conv-04)

<a id="conv-04"></a>
### CONV-04 Single source of truth

- <a id="conv-04-1"></a>**CONV-04.1** A fact (field list, step input or output, threshold, rationale, technology choice) is defined in exactly one item, in the file that owns that item's prefix ([CONV-05](#conv-05)).
- <a id="conv-04-2"></a>**CONV-04.2** Other items reference the fact by link, with at most a short phrase of context. They do not paraphrase, summarise or copy it.
- <a id="conv-04-3"></a>**CONV-04.3** When a fact seems to belong to two items, the most specific owner defines it and the broader item links to it. For example, a step's retry rule belongs to the step, and the general policy links to it.
- <a id="conv-04-4"></a>**CONV-04.4** Rationale lives in decisions ([ADR-01](architecture/decisions.md#adr-01) and onwards). Other items state what is true and link to the ADR for why.
- <a id="conv-04-5"></a>**CONV-04.5** A change to a fact edits only its owner. A review that finds a fact restated elsewhere replaces the copy with a link.

Refs: [CONV-05](#conv-05), [CONV-03](#conv-03)

<a id="conv-05"></a>
### CONV-05 Prefix ownership map

Each prefix is owned by exactly one file. Only that file defines or elaborates items with the prefix.

| Row | Prefix | Item kind | Owning file |
|---|---|---|---|
| <a id="conv-05-1"></a>CONV-05.1 | CONV | Documentation convention | [README.md](README.md) |
| <a id="conv-05-2"></a>CONV-05.2 | GL | Glossary term | [README.md](README.md) |
| <a id="conv-05-3"></a>CONV-05.3 | SCOPE | PoC scope statement or non-goal | [README.md](README.md) |
| <a id="conv-05-4"></a>CONV-05.4 | SC | Success criterion and its acceptance checks | [success-criteria.md](success-criteria.md) |
| <a id="conv-05-5"></a>CONV-05.5 | F | Feature and its sub-capabilities | [features.md](features.md) |
| <a id="conv-05-6"></a>CONV-05.6 | ARCH | Architecture structure and rules | [architecture/overview.md](architecture/overview.md) |
| <a id="conv-05-7"></a>CONV-05.7 | DEP | Runtime component and local deployment | [architecture/overview.md](architecture/overview.md) |
| <a id="conv-05-8"></a>CONV-05.8 | MOD | Source-code module | [architecture/overview.md](architecture/overview.md) |
| <a id="conv-05-9"></a>CONV-05.9 | XC | Cross-cutting concern | [architecture/overview.md](architecture/overview.md) |
| <a id="conv-05-10"></a>CONV-05.10 | DM | Domain entity, value object or aggregate, and its invariants | [architecture/domain.md](architecture/domain.md) |
| <a id="conv-05-11"></a>CONV-05.11 | EVT | Domain event | [architecture/domain.md](architecture/domain.md) |
| <a id="conv-05-12"></a>CONV-05.12 | TAX | Taxonomy or enumeration | [architecture/domain.md](architecture/domain.md) |
| <a id="conv-05-13"></a>CONV-05.13 | CMD | Command | [architecture/application.md](architecture/application.md) |
| <a id="conv-05-14"></a>CONV-05.14 | QRY | Query | [architecture/application.md](architecture/application.md) |
| <a id="conv-05-15"></a>CONV-05.15 | PORT | Inbound or outbound port | [architecture/ports-adapters.md](architecture/ports-adapters.md) |
| <a id="conv-05-16"></a>CONV-05.16 | ADP | Adapter | [architecture/ports-adapters.md](architecture/ports-adapters.md) |
| <a id="conv-05-17"></a>CONV-05.17 | MCP | MCP server aspect, tool or resource | [architecture/mcp.md](architecture/mcp.md) |
| <a id="conv-05-18"></a>CONV-05.18 | WF | Durable workflow definition | [architecture/ingestion.md](architecture/ingestion.md) |
| <a id="conv-05-19"></a>CONV-05.19 | ING | Ingestion pipeline step or contract | [architecture/ingestion.md](architecture/ingestion.md) |
| <a id="conv-05-20"></a>CONV-05.20 | LLM | LLM enrichment prompt contract | [architecture/ingestion.md](architecture/ingestion.md) |
| <a id="conv-05-21"></a>CONV-05.21 | ADR | Architecture decision record | [architecture/decisions.md](architecture/decisions.md) |

Refs: [CONV-01](#conv-01)

## Glossary

Each term is defined only to the depth needed for shared vocabulary. Structure, fields and rules live in the linked owner.

| ID | Term | Definition | Refs |
|---|---|---|---|
| <a id="gl-01"></a>GL-01 | Source | A configured origin of content: a local drive or file-share folder, a SharePoint site or library, or a PST archive. | [DM-01](architecture/domain.md#dm-01) |
| <a id="gl-02"></a>GL-02 | Source item | One object discovered in a source (file, message or attachment). It is tracked so that processing is idempotent and resumable, before and while it becomes a document. | [DM-02](architecture/domain.md#dm-02) |
| <a id="gl-03"></a>GL-03 | Document | The unit of indexing: a file, an e-mail or an attachment, identified by its content hash and carrying metadata, assignments and enrichment. | [DM-03](architecture/domain.md#dm-03) |
| <a id="gl-04"></a>GL-04 | Family | A parent document (an e-mail or a container file) together with its attachments or embedded children. | [DM-03](architecture/domain.md#dm-03) |
| <a id="gl-05"></a>GL-05 | Thread | A reconstructed e-mail conversation that links related messages. | [DM-06](architecture/domain.md#dm-06) |
| <a id="gl-06"></a>GL-06 | Chunk | A token-bounded span of a document's text, indexed for both keyword and vector search. | [DM-05](architecture/domain.md#dm-05) |
| <a id="gl-07"></a>GL-07 | Embedding | A dense vector for a chunk or a query, computed by the external embedding API and stamped with the model version. | [DM-05](architecture/domain.md#dm-05), [ADR-06](architecture/decisions.md#adr-06) |
| <a id="gl-08"></a>GL-08 | Client | A person or organisation the firm acts for. Clients are the top level of organisation. | [DM-07](architecture/domain.md#dm-07) |
| <a id="gl-09"></a>GL-09 | Matter | A case or engagement for a client, with a case type, a responsible lawyer and an access group. | [DM-09](architecture/domain.md#dm-09) |
| <a id="gl-10"></a>GL-10 | Case type | The practice-area classification of a matter, such as employment dispute or M&A. | [TAX-01](architecture/domain.md#tax-01) |
| <a id="gl-11"></a>GL-11 | Handler / responsible lawyer | A person in the firm who is responsible for, or works on, a matter or a document. | [DM-08](architecture/domain.md#dm-08), [TAX-04](architecture/domain.md#tax-04) |
| <a id="gl-12"></a>GL-12 | Assignment | A link from a document to a client, matter, handler or case type, with a method, a confidence and a status. | [DM-10](architecture/domain.md#dm-10) |
| <a id="gl-13"></a>GL-13 | Ethical wall | An access restriction that hides a matter's documents, and the fact that the matter exists, from excluded principals. | [DM-16](architecture/domain.md#dm-16), [XC-01](architecture/overview.md#xc-01) |
| <a id="gl-14"></a>GL-14 | Principal | The authenticated human or agent identity on whose behalf a query or command runs. | [DM-16](architecture/domain.md#dm-16), [PORT-03](architecture/ports-adapters.md#port-03) |
| <a id="gl-15"></a>GL-15 | Enrichment | The attributes the local LLM derives at ingest: document type, tags, case-type suggestion, parties, dates and summary. | [DM-12](architecture/domain.md#dm-12), [LLM-03](architecture/ingestion.md#llm-03) |
| <a id="gl-16"></a>GL-16 | Provenance | A record of how a value was produced (rule, LLM, human or inherited), with version, confidence and actor. | [DM-11](architecture/domain.md#dm-11) |
| <a id="gl-17"></a>GL-17 | Hybrid search / RRF | Search that fuses a keyword (BM25) ranking and a vector ranking with Reciprocal Rank Fusion, under metadata and permission filters. | [QRY-01](architecture/application.md#qry-01), [ADR-03](architecture/decisions.md#adr-03) |
| <a id="gl-18"></a>GL-18 | Ingestion run | One durable, resumable execution of the ingestion workflow over a source scope. | [DM-17](architecture/domain.md#dm-17), [WF-01](architecture/ingestion.md#wf-01) |
| <a id="gl-19"></a>GL-19 | Review task | A queued human decision about an uncertain or conflicting assignment or enrichment. | [DM-13](architecture/domain.md#dm-13) |
| <a id="gl-20"></a>GL-20 | Exact / near duplicate | Exact duplicates share a content hash or an e-mail identity hash. Near duplicates have highly similar text. | [DM-04](architecture/domain.md#dm-04) |

## PoC scope and non-goals

<a id="scope-01"></a>
### SCOPE-01 Local single-machine PoC

- <a id="scope-01-1"></a>**SCOPE-01.1** Everything runs on one host under docker compose. The service set is defined in [DEP-01](architecture/overview.md#dep-01).
- <a id="scope-01-2"></a>**SCOPE-01.2** There are no cloud services, managed databases or Kubernetes. The embedding API is the only external dependency ([SCOPE-04](#scope-04)).
- <a id="scope-01-3"></a>**SCOPE-01.3** The PoC is single-firm and single-tenant.

Refs: [DEP-01](architecture/overview.md#dep-01), [SCOPE-04](#scope-04)

<a id="scope-02"></a>
### SCOPE-02 Sample corpus, 100 GB design

- <a id="scope-02-1"></a>**SCOPE-02.1** The PoC runs end to end on a representative sample, the frozen corpus of [SC-15.1](success-criteria.md#sc-15-1). The sample covers every source and format in [SCOPE-03](#scope-03) and keeps real folder, site and mailbox structure. It contains scans, exact and near duplicates, long e-mail threads, several clients and matters, and at least one walled matter.
- <a id="scope-02-2"></a>**SCOPE-02.2** The 100 GB+ target (about 2M documents and 10M chunks) is proven by design and measurement, not by ingesting 100 GB. The proof is the synthetic scale test in [SC-07](success-criteria.md#sc-07) and the capacity model in [DEP-08](architecture/overview.md#dep-08).
- <a id="scope-02-3"></a>**SCOPE-02.3** A hand-labelled subset of the sample serves as the gold set for evaluation ([F-20](features.md#f-20)).

Refs: [SC-07](success-criteria.md#sc-07), [DEP-08](architecture/overview.md#dep-08), [F-20](features.md#f-20)

<a id="scope-03"></a>
### SCOPE-03 In-scope sources and formats

- <a id="scope-03-1"></a>**SCOPE-03.1** E-mail: PST archives, EML and MSG files, including attachments and nested messages.
- <a id="scope-03-2"></a>**SCOPE-03.2** Office: DOCX, XLSX and PPTX, legacy DOC, XLS and PPT, and RTF.
- <a id="scope-03-3"></a>**SCOPE-03.3** PDF with a text layer, scanned PDF, and scanned images (TIFF, JPEG, PNG).
- <a id="scope-03-4"></a>**SCOPE-03.4** SharePoint Online document libraries with site and column metadata, read live or from an exported folder.
- <a id="scope-03-5"></a>**SCOPE-03.5** Local drive and file-share folders.
- <a id="scope-03-6"></a>**SCOPE-03.6** Out of scope: audio and video, CAD, Teams chats, live Exchange mailboxes (PST only), and content of encrypted or password-protected files. These items are recorded and quarantined with a reason, never dropped silently.

Refs: [F-01](features.md#f-01), [F-02](features.md#f-02), [F-03](features.md#f-03)

<a id="scope-04"></a>
### SCOPE-04 Confidentiality constraint

- <a id="scope-04-1"></a>**SCOPE-04.1** Embeddings are computed by an external API ([ADR-06](architecture/decisions.md#adr-06)), so chunk text and query text leave the machine. The egress controls are in [XC-02](architecture/overview.md#xc-02).
- <a id="scope-04-2"></a>**SCOPE-04.2** The PoC therefore processes only synthetic, public or anonymised data, or matters whose client has explicitly consented to this processing. Real client data outside that set is not loaded.
- <a id="scope-04-3"></a>**SCOPE-04.3** Before the first run with any real data, the embedding provider's terms (no training on inputs, retention, processing region) are checked and recorded.
- <a id="scope-04-4"></a>**SCOPE-04.4** Using the system beyond this data set requires revisiting [ADR-06](architecture/decisions.md#adr-06) (see [ADR-06.5](architecture/decisions.md#adr-06-5)).

Refs: [ADR-06](architecture/decisions.md#adr-06), [XC-02](architecture/overview.md#xc-02)

<a id="scope-05"></a>
### SCOPE-05 Non-goal: answer generation

- <a id="scope-05-1"></a>**SCOPE-05.1** There is no chat, question-answering or answer-synthesis layer. The store returns documents, chunks and metadata.
- <a id="scope-05-2"></a>**SCOPE-05.2** The local LLM is used only for ingest-time enrichment ([ADR-05](architecture/decisions.md#adr-05)). Agents that reason over results are external MCP clients ([F-18](features.md#f-18)).

Refs: [ADR-05](architecture/decisions.md#adr-05), [F-18](features.md#f-18)

<a id="scope-06"></a>
### SCOPE-06 Non-goal: production hardening

- <a id="scope-06-1"></a>**SCOPE-06.1** Excluded: high availability, multi-host failover, backup and disaster recovery, multi-tenant operation, and performance work beyond the targets in [success-criteria.md](success-criteria.md).
- <a id="scope-06-2"></a>**SCOPE-06.2** Excluded: production identity (SSO, OIDC, OAuth 2.1). PoC authentication is described in [ADR-10](architecture/decisions.md#adr-10).
- <a id="scope-06-3"></a>**SCOPE-06.3** Excluded: DMS-grade retention, legal hold, records management and hard-delete propagation.

Refs: [ADR-10](architecture/decisions.md#adr-10)

<a id="scope-07"></a>
### SCOPE-07 Non-goal: write-back to systems of record

- <a id="scope-07-1"></a>**SCOPE-07.1** Originals stay in Exchange, SharePoint and the file share. The store never writes tags, assignments or files back to them, or to the practice-management system.
- <a id="scope-07-2"></a>**SCOPE-07.2** The index and all derived data can be rebuilt from the sources and the reference-data imports. Copies of originals are kept as a cache ([ADR-11](architecture/decisions.md#adr-11)).
- <a id="scope-07-3"></a>**SCOPE-07.3** The exception is data authored in the store: human overrides, review decisions, taxonomy terms and rule sets. These cannot be derived from sources, so they are the only state that would need backup beyond the PoC.

Refs: [ADR-11](architecture/decisions.md#adr-11)

<a id="scope-08"></a>
### SCOPE-08 Non-goal: full CQRS / event sourcing

- <a id="scope-08-1"></a>**SCOPE-08.1** There is no separate read store, no projection synchronisation, no event store and no replay. Commands and queries are separated only lightly, as defined in [ARCH-04](architecture/overview.md#arch-04).
- <a id="scope-08-2"></a>**SCOPE-08.2** Domain events exist only for in-process side effects and the audit log ([ARCH-05](architecture/overview.md#arch-05)).

Refs: [ADR-01](architecture/decisions.md#adr-01), [ARCH-04](architecture/overview.md#arch-04), [ARCH-05](architecture/overview.md#arch-05)
