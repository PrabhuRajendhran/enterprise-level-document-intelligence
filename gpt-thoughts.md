Nice — I’ll do this in two parts in this message:

1. A concise, corrected **architecture outline** that fixes the gaps we discussed and makes explicit the cross-cutting enterprise controls you need.
2. A **final enterprise-ready Mermaid diagram** (paste directly into GitHub Markdown) that implements that outline — clean, phase-grouped, with correct routing, fallbacks, DSPy checks, observability, and model-registry pieces.

---

# 1) Corrected architecture outline (concise, enterprise-ready)

## PHASE 1 — INGESTION & SANITIZATION (THE SHIELD)

* **T0: Unpacker (recursive)**

  * Tools: `libratom`, `unstructured`
  * Actions: Extract attachments, flatten archives → produce files + metadata → enqueue messages.
  * Reliability: Produce events to **Message Bus** (Kafka/SQS); DLQ for bad/unprocessable files.

* **T0.5: Governance Gateway (local CPU)**

  * Spotter: `GLiNER` or `Microsoft Presidio` (PII detection)
  * Classifier: `FastText` (junk/spam)
  * Actions: redact/tag high-sensitivity, reject spam, attach sensitivity metadata.
  * Policy: RBAC + local-only processing for high-sensitivity items (avoid public LLMs).

* **Cross-cutting:** TLS + at-rest encryption, WORM audit log (append-only), ingest metrics (OpenTelemetry).

---

## PHASE 2 — ROUTING CORE (TRAFFIC CONTROLLER)

* **Doc Type Router (T0.5 output)** decides destination using heuristics + lightweight vision checks:

  * Rule 1: Embedded text layer → **Tier 1**
  * Rule 2: Clean scan (regular fonts) → **Tier 2**
  * Rule 3: Complex tables/forms → **Tier 3**
  * Rule 4: Handwriting/unreadable → **Tier 4 (fallback)**

* **Enqueue** chosen processing job to worker pool with priority & cost tags.

---

## PHASE 3 — PROCESSING CORE (LAYOUT & TEXT RECOVERY)

* **Tier 1 — Digital Native**

  * Tools: Docling, MarkItDown
  * Action: Extract internal text & metadata (deterministic).
  * Cost: CPU, near-zero hallucination.

* **Tier 2 — Standard OCR**

  * Tools: Tesseract v5, Mistral Nemo (light correction)
  * Action: Pixel→text; deskew, denoise, script/language detection.

* **Tier 3 — Layout Specialist (post-OCR)**

  * Tool: **Numarkdown-8B** (layout→Markdown/table reconstruction)
  * Important: **Requires Tier 2 output** (text tokens + positional metadata).
  * Action: Reconstruct tables, columns, nested grids into well-formed Markdown.

* **Tier 4 — Visual Reasoning (Nuclear/Fallback)**

  * Tools: Sonnet-4 (vision) / GPT-4o
  * Action: Handwriting, crossed-out text, spatial reasoning. **Invoked only on Tier 2/3 failure or low confidence**.

* **Cross-cutting:** Model version registry, GPU autoscaling, cost controls, SLA tags.

---

## PHASE 4 — SEMANTIC LAYER (MARKDOWN → VALIDATED JSON)

* **Layer A — The Surgeons (Strict Schema)**

  * Models: NuExtract, GoLLIE (primary)
  * Action: Fill known schemas deterministically (invoices, POs). Return structured JSON + confidence per field.

* **Layer B — The Reasoners (Backup)**

  * Models: Sonnet 4, GPT-4o, Qwen 2.5
  * Action: Complex / ambiguous extraction, contract clauses, nested semantics.

* **Layer C — Confidence Gate (DSPy + Rules)**

  * Tools: DSPy assertions, logic scripts, domain validators.
  * Checks: numeric reconciliation (line totals), required fields present, confidence thresholds (>90%).
  * If PASS → ingest to DB/ERP; if FAIL → **Human-in-the-loop UI** (Label Studio) with audit trace.

* **Governance:** PII scrubbed metadata persists; sensitive fields masked in DB if policy requires.

---

## PHASE 5 — CHAT & RETRIEVAL (RAG)

* **Late Chunking**: encode entire cleaned page then chunk (Jina embeddings v3 or in-house).
* **Vector Store**: Qdrant / Weaviate.
* **Hybrid Search**: BM25 (exact IDs) + Vector (semantic).
* **GraphRAG**: link entities across docs; multi-hop reasoning.
* **Reranker → Final LLM Response** with prompt governance and rate limiting.

---

## CROSS-CUTTING ENTERPRISE CONTROLS

* **Message bus & DLQ** for reliability.
* **Observability**: OpenTelemetry traces, Prometheus metrics, Grafana dashboards, alarms.
* **Security**: Zero-trust, OAuth2/JWT, per-resource ACLs, encryption keys (KMS).
* **Audit & Compliance**: Immutable audit logs, per-document provenance, policy engine.
* **CI/CD & Model Registry**: Automated validation suite, canary rollout for model updates (MLflow/Bento).
* **DR & Backups**: Multi-region replication, daily snapshots, RTO/RPO defined.
* **Cost governance**: Model cost tagging, budget alerts, pre-emptive fallback to cheaper models.

---

# 2) Final enterprise Mermaid diagram (paste into GitHub Markdown)

Paste the entire block below into a `.md` file on GitHub (or a GitHub issue/PR). It’s organized by PHASEs, shows queues, DLQ, DSPy checks, Numarkdown placed after OCR, fallbacks, and cross-cutting boxes for observability / security / model registry.

````markdown
```mermaid
flowchart TB
%% ========================================================
%% ENTERPRISE ARCHITECTURE — PHASED, PRODUCTION-READY
%% Paste into GitHub Markdown (Mermaid)
%% ========================================================

%% -------------------------------
%% STYLES
%% -------------------------------
classDef phase fill:#f4f7fb,stroke:#1b66c9,stroke-width:1px,color:#000;
classDef ingest fill:#e9f3ff,stroke:#1b66c9,color:#000;
classDef gov fill:#fff4e6,stroke:#d29b00,color:#000;
classDef processing fill:#eef9f2,stroke:#1f8b73,color:#000;
classDef layout fill:#f7f0ff,stroke:#6c4ea6,color:#000;
classDef reason fill:#fff0f0,stroke:#c44d4d,color:#000;
classDef extract fill:#f6fcff,stroke:#0070c0,color:#000;
classDef db fill:#f0f4f9,stroke:#0b3e7e,color:#000;
classDef cross fill:#f2f2f2,stroke:#666,color:#000;
classDef infra fill:#ffffff,stroke:#333,color:#000;

%% ========================================================
%% PHASE 1 — INGESTION & SANITIZATION (THE SHIELD)
%% ========================================================
subgraph PH1["PHASE 1 — INGESTION & SANITIZATION (The Shield)"]:::phase
direction TB
    Uploader["User Upload / Ingest API"]:::ingest --> UnpackQ["Enqueue: Unpack Job (Message Bus)"]:::ingest

    Unpacker["T0: Unpacker (libratom / unstructured)"]:::ingest -->|extracted files + metadata| SanitizerQ["Enqueue: Governance Scan"]:::ingest
    UnpackQ --> Unpacker

    subgraph GOV["T0.5 — GOVERNANCE GATEWAY (Local CPU)"]:::gov
        direction TB
        Spotter["PII Spotter: GLiNER / Presidio (local)"]:::gov
        Accountant["Junk Classifier: FastText"]:::gov
        Spotter -->|PII detected| Redact["Redact / Tag High Sensitivity"]:::gov
        Accountant -->|Spam / Blank| Reject["Reject (Spam) → DLQ"]:::gov
        Spotter --> SanitizerQ
        Accountant --> SanitizerQ
    end

    SanitizerQ --> GovernanceWorker["Governance Worker (local)"]:::gov
    GovernanceWorker -->|pass| RouterQ["Enqueue: Routing Decision"]:::ingest
    GovernanceWorker -->|reject| DLQ["DLQ (Dead Letter Queue)"]:::infra
end

%% ========================================================
%% PHASE 2 — ROUTING CORE (TRAFFIC CONTROLLER)
%% ========================================================
subgraph PH2["PHASE 2 — ROUTING CORE (Traffic Controller)"]:::phase
direction TB
    Router["Doc Type Router (heuristics + quick vision)"]:::ingest
    RouterQ --> Router

    Router -->|Embedded text layer| Tier1Q["Tier1: Digital Native Queue"]:::processing
    Router -->|Clean scan| Tier2Q["Tier2: Standard OCR Queue"]:::processing
    Router -->|Complex layout (tables/forms)| Tier3Q["Tier3: Layout Specialist Queue"]:::processing
    Router -->|Handwriting / Messy| Tier4Q["Tier4: Visual Reasoning Queue (Fallback)"]:::processing
end

%% ========================================================
%% PHASE 3 — PROCESSING CORE (LAYOUT & TEXT RECOVERY)
%% ========================================================
subgraph PH3["PHASE 3 — PROCESSING CORE (Layout & Text Recovery)"]:::phase
direction TB

    %% Tier1
    subgraph T1["Tier 1 — Digital Native (Deterministic)"]:::processing
        Docling["Docling / MarkItDown"]:::processing
    end
    Tier1Q --> Docling
    Docling --> MarkdownOut["Markdown + Metadata"]:::processing

    %% Tier2
    subgraph T2["Tier 2 — Standard OCR (Pixel→Text)"]:::processing
        Tesseract["Tesseract v5"]:::processing
        NemoFix["Mistral Nemo (corrections)"]:::processing
    end
    Tier2Q --> Tesseract
    Tesseract --> NemoFix
    NemoFix --> OCRText["OCR Text + Positions (tokens)"]:::processing

    %% Tier3 (Layout Specialist - POST OCR)
    subgraph T3["Tier 3 — Layout Specialist (Post-OCR)"]:::layout
        Numark["Numarkdown-8B (layout → Markdown tables)"]:::layout
    end
    Tier3Q -->|If Router detected complex layout -> ensure OCRText exists| OCRText
    OCRText --> Numark
    Numark --> MarkdownOut

    %% Tier4 (Fallback / Reasoning)
    subgraph T4["Tier 4 — Nuclear / Visual Reasoning (Fallback)"]:::reason
        SonnetV["Sonnet-4 (Vision) / GPT-4o"]:::reason
    end
    Tier4Q --> SonnetV
    SonnetV --> MarkdownOut

    %% Failure routing: If Tier1/2/3 produce low confidence -> fallback to Tier4
    MarkdownOut --> ConfidenceCheck1{"Low Level Confidence?"}:::cross
    ConfidenceCheck1 -- yes --> Tier4Q
    ConfidenceCheck1 -- no --> SemanticQ["Enqueue: Semantic Extraction"]:::extract

end

%% ========================================================
%% PHASE 4 — SEMANTIC LAYER (MARKDOWN → VALIDATED JSON)
%% ========================================================
subgraph PH4["PHASE 4 — SEMANTIC LAYER (Extraction → JSON)"]:::phase
direction TB
    SemanticQ --> SurgeonsQ["Layer A — Surgeons Queue (NuExtract / GoLLIE)"]:::extract
    Surgeons["NuExtract / GoLLIE (strict schema)"]:::extract
    SurgeonsQ --> Surgeons
    Surgeons --> JSON["Structured JSON + field confidences"]:::db

    %% If Surgeons low confidence -> Reasoners
    JSON --> ConfGate{"DSPy Checks: totals/sanity/confidence >90%?"}:::cross
    ConfGate -- fail --> HumanLoopQ["Human-in-the-loop (Label Studio)"]:::infra
    ConfGate -- pass --> DBIngestQ["Ingest to DB/ERP (structured)"]:::db

    ConfGate -- partial_fail --> ReasonersQ["Layer B — Reasoners Queue"]:::extract
    Reasoners["Sonnet-4 / GPT-4o / Qwen2.5"]:::extract
    ReasonersQ --> Reasoners
    Reasoners --> JSON2["Revised JSON + confidence"]:::db
    JSON2 --> ConfGate

    HumanLoopQ --> HumanValidate["Human validate / correct"]:::infra
    HumanValidate --> DBIngestQ

    DBIngestQ --> DB["Structured DB / ERP"]:::db

end

%% ========================================================
%% PHASE 5 — CHAT & RETRIEVAL (RAG)
%% ========================================================
subgraph PH5["PHASE 5 — CHAT & RETRIEVAL (RAG)"]:::phase
direction TB
    DB --> LateChunk["Late Chunking & Embedding (Jina v3)"]:::db
    LateChunk --> VectorDB["Vector DB (Qdrant/Weaviate)"]:::db
    UserQuery["User Query"]:::processing --> SearchRouter{"IDs / Field Filter / Complex Logic?"}:::cross
    SearchRouter -->|IDs exact| BM25["BM25 (Keyword index)"]:::db
    SearchRouter -->|semantic| VectorDB
    VectorDB --> HybridRank["Hybrid Rank (BM25 + Vector)"]:::db
    HybridRank --> Reranker["Reranker"]:::extract
    Reranker --> FinalLLM["Final LLM Response (prompt governance)"]:::reason
    FinalLLM --> UserOutput["User-facing Response (UI)"]:::infra
    %% GraphRAG
    VectorDB --> GraphRAG["GraphRAG: entity linking & multi-hop"]:::db
    GraphRAG --> Reranker
end

%% ========================================================
%% CROSS-CUTTING: INFRA, OBSERVABILITY, SECURITY, MODEL REGISTRY
%% ========================================================
subgraph CROSS["CROSS-CUTTING SERVICES"]:::phase
direction LR
    Observability["Observability: OpenTelemetry / Prometheus / Grafana"]:::cross
    AuthN["AuthN/AuthZ: OAuth2 / JWT / RBAC"]:::cross
    Audit["Immutable Audit Log / Provenance (append-only)"]:::cross
    ModelReg["Model Registry & CI: MLflow / Canary + Versioning"]:::cross
    QueueSys["Message Bus: Kafka / SQS + DLQ"]:::cross
    Secrets["KMS / Secrets / Data Encryption"]:::cross
    DR["DR & Backups: multi-region snapshots"]:::cross

    %% linking visuals
    UnpackQ -.-> QueueSys
    RouterQ -.-> QueueSys
    Tier1Q -.-> QueueSys
    Tier2Q -.-> QueueSys
    Tier3Q -.-> QueueSys
    Tier4Q -.-> QueueSys
    SurgeonsQ -.-> QueueSys
    ReasonersQ -.-> QueueSys
    HumanLoopQ -.-> QueueSys
    DBIngestQ -.-> QueueSys

    %% Observability & Audit taps
    Observability -.-> Unpacker
    Observability -.-> GovernanceWorker
    Observability -.-> Docling
    Observability -.-> Numark
    Observability -.-> Surgeons
    Observability -.-> Reasoners
    Audit -.-> DB
    ModelReg -.-> Numark
    ModelReg -.-> SonnetV
    ModelReg -.-> Surgeons
end

%% ========================================================
%% LEGEND / NOTES
%% ========================================================
note right of CROSS
 - DSPy = Domain-specific assertions (numeric checks, schema validation)
 - Numarkdown is POST-OCR layout reconstructor (requires OCR tokens & positions)
 - Tier4 invoked only on failure / low-confidence or handwriting
 - Sensitive docs flagged redacted before any public LLM calls
endnote

```
````

---

If you want, next I can:

* Convert this into a one-page **presentation-ready PNG/SVG** (I can generate a high-res SVG-friendly Mermaid that you can drop into slides).
* Produce a compact **implementation checklist** (queues, DLQ, KMS, observability dashboards, test dataset).
* Produce a **model routing table** that includes costs, expected latency, and suggested autoscaling rules.

Which do you want next?
