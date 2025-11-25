Nice — I’ll deliver everything in one go: a precise, production-ready **updated architecture outline** with detailed rationales, an explicit **routing logic** (rules + heuristics you can implement), a clean **enterprise Mermaid diagram** you can paste into GitHub Markdown, and short **remarks** explaining why this architecture is superior to a “model-only” stacking approach.

# 1) Updated architecture outline — phases, tiers, steps, and why each exists

> High-level goal: maximize accuracy and throughput while minimizing cost, risk, and human review. Each phase has a single responsibility and explicit failure/fallback behaviors.

## PHASE 1 — INGESTION & SANITIZATION (THE SHIELD)

**Goal:** Accept files reliably, unpack, classify sensitivity, and avoid sending PII to public models.

* **T0 — Unpacker (recursive)**

  * Tools: `libratom`, `unstructured`
  * Action: extract attachments (.zip/.msg/.eml/.tar), normalize inputs (PDF, image, docx, xlsx), attach file metadata (MIME, size, page count).
  * Rationale: ~40% of useful content lives in attachments; failing to unpack loses data and causes downstream "binary garbage" errors.
  * Failure handling: bad archives → DLQ; metadata stored for observability.

* **T0.5 — Governance Gateway (local CPU)**

  * Tools: `GLiNER` or `Microsoft Presidio` (for PII spotting), `FastText` (spam/junk classifier).
  * Action: tag/redact PII (SSN, CC), reject spam/blank pages, attach `sensitivity_level` metadata.
  * Rationale: regulatory compliance (GDPR/SOC2) and cost avoidance (don’t pay for LLM on spam). Local CPU mitigates data-exfil risk.
  * Failure handling: high-sensitivity docs are marked for local-only processing or human-review; spam → DLQ.

**Cross-cutting:** TLS, storage encryption (KMS), append-only audit log for provenance, ingestion metrics & traces (OpenTelemetry).

---

## PHASE 2 — ROUTING CORE (THE TRAFFIC CONTROLLER)

**Goal:** Route each document to the *cheapest* component that can solve it with sufficient confidence.

* **Doc Type Router (hybrid heuristics + tiny vision checks)**

  * Inputs: file metadata, quick page-scan summary (DPI, color, page-count), optional embedded text presence.
  * Output: route to Tier1 / Tier2A / Tier2B / Tier3 / Tier4 + a `priority` tag and `cost_budget` hint.

**Why:** routing early avoids unnecessary expensive inference; reduces latency and cost at scale.

---

## PHASE 3 — PROCESSING CORE (OCR → LAYOUT → CLEAN TEXT)

**Goal:** Produce *clean*, position-aware Markdown/Text that downstream extractors can consume.

**Design principle:** separate concerns — (A) pixel→text, (B) text+positions→layout reconstruction.

* **Tier 1 — Digital Native (Deterministic)**

  * Tools: `Docling`, `MarkItDown`
  * Input: born-digital (PDF with text, DOCX, XLSX)
  * Output: text + precise structural metadata (tables, form fields).
  * Rationale: deterministic, zero hallucination, near-zero cost. Use whenever embedded text exists.

* **Tier 2A — Fast OCR (Cheap baseline)**

  * Tools: `Tesseract v5` / `OCRmyPDF` / `PaddleOCR`
  * Use case: high-quality scans, high DPI, standard fonts.
  * Output: text tokens + bounding boxes + line-grouping.
  * Rationale: extremely cost-efficient; handles ~60–80% of documents.

* **Tier 2B — Intelligent OCR (Robust fallback)**

  * Tools: `Surya (full)`, `Mistral-OCR`, advanced OCR APIs
  * Use case: noisy scans, skewed pages, mixed encodings, some handwriting.
  * Output: higher-quality tokens + more accurate bboxes & reading order.
  * Rationale: run only when Fast OCR fails quality checks (low confidence, poor bbox overlap), or routing flags complex layout/noise.

* **Tier 3 — Layout-Aware (Post-OCR Form Specialist)**

  * Tools: `Numarkdown-8B` *or* other layout reconstructor
  * Input: OCR tokens + bounding boxes + reading order (from Tier1 or Tier2A/B)
  * Output: well-formed Markdown (preserves table cells, nested grids, column grouping).
  * Rationale: fixes layout semantics so downstream structured extractors see tabular data rather than flattened lines. **Numarkdown is a post-OCR model — it cannot replace Tier 2.**

* **Tier 4 — VLM & Long-Context Reasoning (Nuclear / Fallback)**

  * Models: `Sonnet-4 (vision)`, `GPT-4o` , `Gemini 1.5 Pro` (for long-context)
  * Use case: handwriting, heavily degraded scans, cross-page tables, ambiguous spatial reasoning.
  * Action: perform pixel-level recognition + spatial reasoning; can produce final Markdown if Tier2+3 cannot.
  * Rationale: expensive — use sparingly; reserved for documents where automated pipeline cannot meet confidence thresholds.

**Failure & fallback:** If Tier2A output fails quality checks → route to Tier2B. If Tier2B fails layout checks or Tier3 cannot produce valid Markdown → Tier4. All steps produce quality/confidence scores.

---

## PHASE 4 — SEMANTIC LAYER (MARKDOWN → VALIDATED JSON)

**Goal:** Convert clean Markdown/text into high-fidelity, schema-validated JSON.

* **Layer A — The Surgeons (Strict Schema)**

  * Tools: `NuExtract`, `GoLLIE`
  * Use case: predictable documents (invoices, POs, remittance advices) with known fields.
  * Output: JSON with per-field confidence and provenance (source token ranges).
  * Rationale: fine-tuned extractors minimize hallucination; deterministic mapping preferred.

* **Layer B — The Reasoners (Backup / Complex)**

  * Tools: `Claude Sonnet 4`, `GPT-4o`, `Qwen 2.5`
  * Use case: long contracts, ambiguous fields, nested semantics, semantic inference.
  * Action: invoked when Surgeons fail or field confidence < threshold.

* **Layer C — Confidence Gate (DSPy + Rules)**

  * Tools: `DSPy assertions`, business logic scripts (sums, tax checks, required-field checks)
  * Rule examples: `total == sum(line_items)`, `invoice_date in range`, `mandatory_fields present`
  * Pass → DB/ERP ingest. Fail → Human-in-the-loop (Label Studio) with task context and correction UI.
  * Rationale: prevents bad data poisoning downstream systems.

---

## PHASE 5 — CHAT & RETRIEVAL (RAG)

**Goal:** Allow safe, accurate, context-rich querying and chat over ingested data.

* **Late chunking & embeddings:** encode cleaned pages first (Jina v3 or in-house embeddings), then chunk to preserve context.
* **Vector store:** `Qdrant` / `Weaviate`.
* **Hybrid search:** BM25 (exact IDs) + vector retrieval (semantic).
* **GraphRAG:** entity linking for multi-document reasoning (Vendor→Invoices→Contracts).
* **Reranker & final LLM:** constrained prompts, retrieval-augmented generation with prompt governance and rate limits.

---

## CROSS-CUTTING CONTROLS (must be explicit)

* **Message bus & DLQ:** `Kafka` / `SQS` for scaling and retry.
* **Model registry & CI:** `MLflow`, canary rollouts, model version pins in pipeline.
* **Observability:** OpenTelemetry traces, Prometheus metrics, Grafana alerts (fail rates, latency per tier).
* **Security:** OAuth2/JWT, per-resource RBAC, KMS for keys, VPC / private-only routing for sensitive docs.
* **Audit & Compliance:** append-only provenance logs, per-document processing chain.
* **Cost governance:** model call budgets, autoscaling thresholds, fallbacks to cheaper models.

---

# 2) Routing logic — explicit rules, heuristics, and confidence thresholds

> Implement as a small service triggered after T0.5; it returns a routing decision and `cost_budget` tag.

## Core rules (deterministic order)

1. **Rule 1 — Digital Native**

   * Condition: PDF has text layer OR DOCX/XLSX/HTML.
   * Action: Route → Tier 1 (Docling). `cost_budget = low`.
   * Reason: deterministic extraction.

2. **Rule 2 — Fast OCR (Tier 2A)**

   * Condition: image DPI ≥ 200, skew ≤ 2°, color channels OK, no handwriting detected, single-column layout heuristics pass.
   * Action: Route → Tier 2A (Tesseract). `cost_budget = very_low`.
   * Post-check: compute `ocr_quality_score` (e.g., average character confidence, token density, bbox overlap). If `ocr_quality_score < q1` → send to Tier 2B.

3. **Rule 3 — Complex Layout (Tier 2B → Tier 3)**  ← corrected behavior

   * Condition: table / multi-column / form detected in quick layout scan OR detected by heuristic (large number of vertical alignment candidates, repetitive numeric columns).
   * Action: Route → Tier 2B (Surya / Mistral-OCR) **first** to produce tokens + bboxes → then automatically enqueue Tier 3 (Numarkdown-8B) for layout reconstruction. `cost_budget = medium`.
   * Note: never send raw pixels directly to Numarkdown.

4. **Rule 4 — Messy / Handwriting / Spatial Reasoning (Tier 4)**

   * Condition: handwriting detected, very low OCR quality, rotated/skewed > 10°, or document flagged as business-critical and low confidence from previous tiers.
   * Action: Route → Tier 4 (Sonnet-4 / GPT-4o). `cost_budget = high` and `priority = high`.
   * Note: Tier 4 can perform direct image→text + layout reasoning; consider human-in-the-loop thresholds.

## Heuristics & thresholds (example numeric values you can tune)

* `ocr_quality_score` from 0–100

  * if ≥ 85 → accept Tier2A output
  * if 60–85 → escalate to Tier2B
  * if < 60 → escalate to Tier4
* `table_likelihood_score` (0–1) — derived from column alignment votes

  * if > 0.6 → treat as complex layout
* `handwriting_confidence`

  * if > 0.3 → consider Tier4
* `sensitivity_level` (HIGH/LOW) from T0.5

  * if HIGH → restrict routing to local-only models (no public cloud LLMs)

## Suggested implementation details

* The router should be a stateless microservice with small vision heuristics (fast), not a full VLM.
* Store router decision + reason for audit.
* Allow manual override and replay.

---

# 3) Final enterprise Mermaid diagram (paste directly into GitHub Markdown)

````markdown
```mermaid
flowchart TB
%% ========================================================
%% FINAL ENTERPRISE ARCHITECTURE — PHASED, PRODUCTION-READY
%% Paste into GitHub Markdown (Mermaid)
%% ========================================================

%% Styles
classDef phase fill:#f4f7fb,stroke:#1b66c9,stroke-width:1px,color:#000;
classDef ingest fill:#e9f3ff,stroke:#1b66c9,color:#000;
classDef gov fill:#fff4e6,stroke:#d29b00,color:#000;
classDef processing fill:#eef9f2,stroke:#1f8b73,color:#000;
classDef layout fill:#f7f0ff,stroke:#6c4ea6,color:#000;
classDef reason fill:#fff0f0,stroke:#c44d4d,color:#000;
classDef extract fill:#f6fcff,stroke:#0070c0,color:#000;
classDef db fill:#f0f4f9,stroke:#0b3e7e,color:#000;
classDef cross fill:#f2f2f2,stroke:#666,color:#000;

%% PHASE 1 — INGESTION & SANITIZATION
subgraph PH1["PHASE 1 — INGESTION & SANITIZATION (The Shield)"]:::phase
  direction TB
  Uploader["User Upload / API"]:::ingest --> EnqueueUnpack["Enqueue Unpack Job (Message Bus)"]:::ingest
  EnqueueUnpack --> Unpacker["T0: Unpacker (libratom / unstructured)"]:::ingest
  Unpacker --> GovQueue["Enqueue Governance Scan"]:::ingest

  subgraph GOV["T0.5 — Governance Gateway (Local CPU)"]:::gov
    GLiNER["PII Spotter: GLiNER / Presidio"]:::gov
    FastText["Junk Classifier: FastText"]:::gov
    GLiNER -->|PII detected| Redact["Redact / Tag High Sensitivity"]:::gov
    FastText -->|Spam/Blank| Reject["Reject → DLQ"]:::gov
    Redact --> GovernanceWorker["Governance Worker"]:::gov
    Reject --> DLQ["DLQ / Quarantine"]:::cross
  end

  GovQueue --> GovernanceWorker
  GovernanceWorker -->|pass| RouterQueue["Enqueue Router Decision"]:::ingest
  GovernanceWorker -->|reject| DLQ
end

%% PHASE 2 — ROUTING CORE
subgraph PH2["PHASE 2 — ROUTING CORE (Traffic Controller)"]:::phase
  direction TB
  Router["Doc Type Router (heuristics + quick vision)"]:::ingest
  RouterQueue --> Router

  Router -->|embedded text| Tier1Q["Tier1 Queue (Digital Native)"]:::processing
  Router -->|clean scan| Tier2AQ["Tier2A Queue (Fast OCR)"]:::processing
  Router -->|complex layout| Tier2BQ["Tier2B Queue (Intelligent OCR) -> then Tier3"]:::processing
  Router -->|handwriting/messy| Tier4Q["Tier4 Queue (VLM / Nuclear)"]:::processing
end

%% PHASE 3 — PROCESSING CORE
subgraph PH3["PHASE 3 — PROCESSING CORE (OCR → Layout → Markdown)"]:::phase
  direction TB

  %% Tier1: Digital Native
  Tier1Q --> Docling["Tier1: Docling / MarkItDown (embed text)"]:::processing
  Docling --> MarkdownOut["Markdown + Positions (clean)"]:::processing

  %% Tier2A: Fast OCR
  Tier2AQ --> Tesseract["Tier2A: Tesseract v5 (Fast OCR)"]:::processing
  Tesseract --> OCRTextA["OCR tokens + bboxes (quality score)"]:::processing

  %% Tier2B: Intelligent OCR
  Tier2BQ --> Surya["Tier2B: Surya / Mistral-OCR (Robust OCR)"]:::processing
  Surya --> OCRTextB["OCR tokens + bboxes (high-quality)"]:::processing

  %% Tier3: Layout Specialist (POST-OCR)
  OCRTextA -->|if quality low or table_likelihood high| Surya
  OCRTextB --> Numark["Tier3: Numarkdown-8B (Layout → Markdown)"]:::layout
  OCRTextA --> Numark
  Numark --> MarkdownOut

  %% Tier4: VLM / Long Context (Fallback)
  Tier4Q --> VLM["Tier4: Sonnet-4 / GPT-4o / Gemini (image→text+layout)"]:::reason
  VLM --> MarkdownOut

  %% Quality check
  MarkdownOut --> LowLevelConfidence{"Low level confidence?"}:::cross
  LowLevelConfidence -- yes --> Tier4Q
  LowLevelConfidence -- no --> SemanticQueue["Enqueue Semantic Extraction"]:::extract
end

%% PHASE 4 — SEMANTIC LAYER (Markdown → JSON)
subgraph PH4["PHASE 4 — SEMANTIC LAYER (Extraction → JSON)"]:::phase
  direction TB
  SemanticQueue --> SurgeonsQ["Layer A: Surgeons Queue (NuExtract / GoLLIE)"]:::extract
  SurgeonsQ --> Surgeons["NuExtract / GoLLIE (strict schema)"]:::extract
  Surgeons --> JSONOut["Structured JSON + field confidences"]:::db

  JSONOut --> DSPyGate{"DSPy Checks: totals/sanity/confidence > 90%?"}:::cross
  DSPyGate -- pass --> DBIngestQ["Ingest to DB/ERP"]:::db
  DSPyGate -- fail --> HITLQ["Human-in-the-loop (Label Studio)"]:::cross
  DSPyGate -- partial --> ReasonersQ["Layer B: Reasoners Queue"]:::extract

  ReasonersQ --> Reasoners["Sonnet-4 / GPT-4o / Qwen2.5"]:::extract
  Reasoners --> JSON2["Revised JSON + confidence"]:::db
  JSON2 --> DSPyGate
  HITLQ --> HumanValidate["Human Validate / Correct"]:::cross
  HumanValidate --> DBIngestQ
  DBIngestQ --> DB["Structured DB / ERP"]:::db
end

%% PHASE 5 — CHAT & RETRIEVAL (RAG)
subgraph PH5["PHASE 5 — CHAT & RETRIEVAL (RAG)"]:::phase
  direction TB
  DB --> LateChunk["Late Chunking & Embeds (Jina v3)"]:::db
  LateChunk --> VectorDB["Vector DB (Qdrant / Weaviate)"]:::db
  UserQuery["User Query"]:::processing --> SearchRouter{"IDs / Field Filter / Complex Logic?"}:::cross
  SearchRouter -->|IDs exact| BM25["BM25 (keyword)"]:::db
  SearchRouter -->|semantic| VectorDB
  VectorDB --> HybridRank["Hybrid Rank (BM25 + Vector)"]:::db
  HybridRank --> Reranker["Reranker"]:::extract
  Reranker --> FinalLLM["Final LLM (prompt governance)"]:::reason
  FinalLLM --> UserFacing["User Response (UI)"]:::cross
  VectorDB --> GraphRAG["GraphRAG (entity linking / multi-hop)"]:::db
  GraphRAG --> Reranker
end

%% CROSS-CUTTING SERVICES
subgraph CROSS["CROSS-CUTTING — Infra, Observability, Security, Model Registry"]:::phase
  direction LR
  Observability["OpenTelemetry / Prometheus / Grafana"]:::cross
  AuthN["OAuth2 / JWT / RBAC / VPC"]:::cross
  Audit["Append-only Audit Log / Provenance"]:::cross
  ModelReg["Model Registry / MLflow / Canary"]:::cross
  QueueSys["Message Bus: Kafka / SQS / DLQ"]:::cross
  Secrets["KMS / Secrets"]:::cross

  %% Links for visibility
  EnqueueUnpack -.-> QueueSys
  RouterQueue -.-> QueueSys
  Tier1Q -.-> QueueSys
  Tier2AQ -.-> QueueSys
  Tier2BQ -.-> QueueSys
  Tier3 -.-> QueueSys
  Tier4Q -.-> QueueSys
  SurgeonsQ -.-> QueueSys
  ReasonersQ -.-> QueueSys

  Observability -.-> Unpacker
  Observability -.-> GovernanceWorker
  Observability -.-> Numark
  Observability -.-> Surgeons
  Audit -.-> DB
  ModelReg -.-> Numark
  ModelReg -.-> VLM
end

note right of CROSS
 - DSPy = domain assertions (line totals, required fields, numeric sanity)
 - Numarkdown is POST-OCR: requires tokens + bbox positions
 - Tier4 invoked only as fallback or for handwriting / spatial inference
 - Sensitive docs are processed local-only (no public LLM calls)
endnote
```
````

---

# 4) Remarks — Why this architecture is better & why “just stacking models” won’t work

## Why this design is better (concise)

1. **Single responsibility & modularity:** each phase has one job (ingest, sanitize, route, OCR, layout, extract, verify, store). That reduces brittle interdependence.
2. **Cost-efficiency:** cheap deterministic paths (Tier1/Tier2A) handle the majority, expensive models used sparingly (Tier4).
3. **Higher accuracy:** layout-aware reconstruction (Numarkdown post-OCR) dramatically improves extraction accuracy and reduces HITL.
4. **Safety & compliance:** local PII filter, RBAC, audit logs prevent data leakage and regulatory exposure.
5. **Resilience & observability:** queues, DLQ, metrics and model registry enable safe rollouts and rapid rollback.
6. **Deterministic validation:** DSPy checks prevent garbage ingestion into ERP/DB.

## Why simply stacking your leader’s models (naïve stacking) won’t help

1. **Wrong tool for wrong role:** models like Numarkdown, GLiNER, GoLLIE are specialized — simply calling them in sequence on raw images will fail. Numarkdown needs OCR tokens; GLiNER is for NER/PII not schema mapping.
2. **Cost blowout:** calling VLMs (GPT-4o/Sonnet/Gemini) on every document is massively expensive; routing and tiering avoids unnecessary spend.
3. **Conflicting outputs & bounding boxes:** running multiple OCRs and feeding different token orders into Numarkdown/GoLLIE leads to inconsistent provenance and harder debugging.
4. **Lack of deterministic checks:** stacking without DSPy or reconciliation means hallucinated fields will silently pollute downstream systems.
5. **Operations & compliance risk:** not having local PII filter + audit trail exposes you to legal risk and vendor lock-in.
6. **No model governance:** naive stacks don’t include model registry, versioning, canarying — you will get silent accuracy regressions.

---

# Next steps I can do for you (pick any/all)

* Produce a **compact implementation checklist** (queues, DLQ, KMS, endpoints, example payloads).
* Create a **model routing table** with example thresholds & monitoring alerts.
* Generate a **one-slide SVG** of the Mermaid diagram for presentations.
* Produce sample **DSPy rules** for invoice sanity checks.

Which one should I do next?
