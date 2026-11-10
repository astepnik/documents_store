# documents_store

Research and recommendation for an AI-friendly document store for a law firm
(100 GB+ of e-mail, SharePoint and file-share content, indexed by client,
handler and case type, searchable by humans and AI agents).

- **PoC design docs:** [`docs/README.md`](docs/README.md) is the entry point for the success criteria, the feature catalogue and the hexagonal architecture map. Every item has a stable, linkable ID.
- **Report:** [`reports/Law firm AI document store options.md`](reports/Law%20firm%20AI%20document%20store%20options.md)
- **Research notes** (one file per track, with inline sources):
  `research_notes/Law firm AI document store options/`
  - `open_source_engines_and_dms.md` – search/vector engines and open-source DMSs
  - `open_source_ingestion_and_rag.md` – parsers, connectors, Onyx/RAGFlow and friends, local embedding models
  - `legal_vertical_saas.md` – Libra by Wolters Kluwer, NetDocuments, iManage, legal AI layers
  - `horizontal_saas_and_cloud.md` – Microsoft 365/Copilot, Azure AI Search, Glean, AWS, Google, M-Files
  - `build_your_own_architecture.md` – reference architecture, sizing, hardware, effort
  - `compliance_and_case_studies.md` – GDPR, professional secrecy (PL/DE/UK/EU), AI Act, firm case studies
