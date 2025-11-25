Here is the fully consolidated, production-ready **TITAN-IDP 2.0 (The "Redemption" Architecture)**.

This version incorporates the **Governance Layer** (to fix the legal risks), the **Cost-Routing Logic** (to fix the bankruptcy risks), and the **Tiered Model Strategy** (to fix the operational naivety), formatted exactly as requested.

***

# 🏛️ TITAN-IDP 2.0: Enterprise Document Intelligence Engine
**Mission:** Ingest any file, **secure it**, route it intelligently, and extract precise data with **zero-trust validation**.

---

### 🔷 PHASE 1: INGESTION & SANITIZATION (The "Shield")
**Goal:** Receive files, unpack them, and **scrub sensitive data** before expensive AI touches them.

#### TIER 0: THE UNPACKER (Recursive Utilities)
* **What it is:** Recursive script for zips, emails (.msg/.eml), and tarballs.
* **Tools:** `libratom`, `unstructured`.
* **Why it matters:** 40% of data is hidden in attachments.
* **What fails if missing:** The system treats a zip file as binary garbage.

#### 🛑 TIER 0.5: THE GOVERNANCE GATEWAY (The "Legal Saver") 🆕
* **What it is:** A local CPU-based filter that scans for PII and "Junk" before processing.
* **Tools:**
    * **The Spotter:** `GLiNER` or `Microsoft Presidio` (Local CPU).
    * **The Accountant:** `FastText` classifier.
* **Action:**
    1.  **PII Check:** If SSN/Credit Card detected → Redact or Tag "High Sensitivity".
    2.  **Junk Check:** If file = "Spam" or "Blank" → Reject immediately.
* **Why it matters:** Prevents sending customer PII to public LLMs (GDPR/SOC2 compliance) and stops you from paying $0.05 to process a blank page.
* **What fails if missing:** You violate privacy laws and go bankrupt processing spam.

---

### 🔷 PHASE 2: THE ROUTING CORE (The "Traffic Controller")
**Goal:** Send the document to the **cheapest possible model** that can solve the problem.

#### THE ROUTER LOGIC
* **Rule 1 (Digital Native):** If PDF has embedded text layer → **Go to Tier 1**.
* **Rule 2 (Clean Scan):** If standard font & simple layout → **Go to Tier 2**.
* **Rule 3 (Complex Layout):** If tables, forms, or mixed columns → **Go to Tier 3**.
* **Rule 4 (Messy/Handwriting):** If unreadable, handwritten, or requires reasoning → **Go to Tier 4**.

---

### 🔷 PHASE 3: THE PROCESSING CORE (Layout & Text Recovery)
**Goal:** Convert visual pixels into structured Markdown.

#### TIER 1: DIGITAL NATIVE (The "Truth" Layer)
* **Target:** Born-digital PDFs, Word, Excel, HTML.
* **Model:** `Docling` (IBM) or `MarkItDown` (Microsoft).
* **Why it matters:** Deterministic. It reads the internal code. **Zero Hallucination Risk.**
* **Cost:** Near Zero (CPU only).

#### TIER 2: STANDARD OCR (The "Commodity" Layer)
* **Target:** Clean scans, simple letters.
* **Model:** `Tesseract 5` or `Mistral Nemo 12B` (for light correction).
* **Why it matters:** Handles 60% of business docs cheaply.
* **Cost:** Very Low.

#### TIER 3: THE SPECIALIST (The "Layout" Layer) 🆕
* **Target:** Complex Invoices, Tables, Bank Statements.
* **Model:** `Numarkdown-8b`.
* **Why it matters:** This model specializes in converting visual grids into Markdown tables without breaking the structure. It replaces generic OCR for complex formatting.
* **What fails if missing:** Tables are read as flat text, destroying row/column relationships.

#### TIER 4: THE NUCLEAR OPTION (The "Reasoning" Layer)
* **Target:** Handwriting, Crossed-out text, "Spatial" reasoning.
* **Model:** `Claude Sonnet 4` (Visual King) or `GPT-4o`.
* **Why it matters:** Can read "between the lines" (e.g., a handwritten note in the margin).
* **Cost:** High (Use sparingly).

---

### 🔷 PHASE 4: THE SEMANTIC LAYER (Extraction to JSON)
**Goal:** Turn Markdown into validated JSON Data using the **"Surgeon vs. Reasoner"** strategy.

#### LAYER A: THE SURGEONS (Strict Schema) 🆕
* **Target:** Standard Forms (Invoices, POs) where fields are known.
* **Models:** `NuExtract 1.5` (Standard) or `GoLLIE` (Python-heavy).
* **Why it matters:** These are fine-tuned to **force** text into a schema. They hallucinate less than Chat models because they are trained for "Pure Extraction."

#### LAYER B: THE REASONING BACKUP (Complex/Messy)
* **Target:** Legal contracts, long unstructured paragraphs.
* **Models:** `Claude Sonnet 4` (Cloud) or `Qwen 2.5` (Local/Private).
* **Action:** Invoked only if `NuExtract` fails or returns low confidence.
* **Why it matters:** Handles nuance that strict extractors miss.

#### LAYER C: THE CONFIDENCE GATE (The "Safety Net") 🆕
* **Tool:** `DSPy` assertions + Logic Scripts.
* **Action:**
    * Check: `Total` == `Sum(Line Items)`?
    * Check: `Confidence Score` > 90%?
* **Result:**
    * **Pass:** Send to Database/ERP.
    * **Fail:** **Route to Human-in-the-Loop UI (Label Studio).**
* **What fails if missing:** Bad data pollutes your downstream systems silently.

---

### 🔷 PHASE 5: THE CHAT & RETRIEVAL ENGINE (RAG)
**Goal:** Chat with the data *after* it has been cleaned and structured.

#### STEP 1: LATE CHUNKING
* **Model:** `Jina AI Embeddings v3`.
* **Why it matters:** Preserves context by encoding the whole page before cutting it up.

#### STEP 2: HYBRID SEARCH
* **Database:** `Qdrant` or `Weaviate`.
* **Strategy:** Vector (Concepts) + Keyword (BM25 for exact ID matches).

#### STEP 3: GRAPHRAG (Knowledge Graph)
* **Why it matters:** Links entities across documents (e.g., Vendor X -> Invoice A -> Contract B).


Flowchart :

```mermaid
flowchart TD

%% =========================================================
%% ENTERPRISE COLOR STYLES
%% =========================================================
classDef ingest fill:#e8f1fb,stroke:#1b66c9,stroke-width:1px,color:#000;
classDef router fill:#fff4d6,stroke:#d29b00,stroke-width:1px,color:#000;
classDef model fill:#e5f7f3,stroke:#1f8b73,stroke-width:1px,color:#000;
classDef gateway fill:#fdeaea,stroke:#c44d4d,stroke-width:1px,color:#000;
classDef db fill:#dce9f7,stroke:#0b3e7e,stroke-width:1px,color:#000;
classDef output fill:#f2f2f2,stroke:#555,stroke-width:1px,color:#000;

%% =========================================================
%% SECTION 1 — INGESTION PIPELINE
%% =========================================================
subgraph INGEST["📥 ENTERPRISE INGESTION PIPELINE"]
direction TB

A["1. User Uploads File"]:::ingest --> B{"1.1 Universal Router"}:::router

B -->|"Email / Archive"| C["1.2 T0: Unpacker (libratom)"]:::ingest
B -->|"PDF / Office / Binary"| B1

C -->|"Body Text / Doc"| B1
C -->|"Attachments"| B

B -->|"Structured Data"| D["1.3 Data Agent"]:::ingest
D --> E["1.4 Pandas Engine"]:::ingest
E --> F["Exact Structured JSON Output"]:::output
end

%% =========================================================
%% SECTION 2 — GOVERNANCE & COMPLIANCE
%% =========================================================
subgraph GOV["🛡 GOVERNANCE, RISK & COMPLIANCE"]
direction TB

B1{"2.0 Governance Gateway"}:::gateway
B1 -->|"Junk / Spam"| Z["Z: REJECT"]:::output
B1 -->|"PII Scrubbed (GLiNER)"| G1{"2.1 Doc Type Router"}:::router

G1 -->|"Digital Native"| G["3.1 T1: Docling"]:::model
G1 -->|"Scanned Clean"| L["3.2 T2: OCR (Tesseract)"]:::model
G1 -->|"Complex Forms / Tables"| M["3.3 T3: Layout Specialist"]:::model
G1 -->|"Messy / Handwritten"| K["3.4 T4: VLM Reasoning"]:::model
end

%% =========================================================
%% SECTION 3 — SEMANTIC EXTRACTION ENGINE
%% =========================================================
subgraph EXTRACT["🧠 ENTERPRISE SEMANTIC EXTRACTION"]
direction TB

G --> J["Markdown Output"]:::output
L --> J
M --> J
K --> J

J --> J1["4.1 Semantic Extractor"]:::model

J1 -->|"Strict Schema"| S1["Surgeon Models (NuExtract / GoLLIE)"]:::model
J1 -->|"Reasoning Required"| S2["Reasoner Models (Sonnet4 / Qwen)"]:::model

S1 --> J2{"4.2 Confidence Gate"}:::gateway
S2 --> J2

J2 -->|"PASS > 90%"| DB["4.4 Structured DB / ERP"]:::db
J2 -->|"FAIL < 90%"| J3["4.3 Human Review UI"]:::output
J3 -->|"Validated"| DB

F --> DB
end

%% =========================================================
%% SECTION 4 — SEARCH + RAG STRATEGY
%% =========================================================
subgraph RAG["🔎 SEARCH & RETRIEVAL (RAG)"]
direction TB

DB --> R["Metadata Filter"]:::ingest

User["User Query"]:::ingest --> P{"5.2 Search Strategy"}:::router

P -->|"IDs / Exact Codes"| Q["Hybrid Search (BM25 + Vector)"]:::ingest
P -->|"Field Filter"| R
P -->|"Complex Logic"| S["GraphRAG (Multi-hop)"]:::model

N["5.1 Embeddings / Chunking"]:::model --> O["Vector DB"]:::db
J --> N

O --> Q
O --> S

Q --> T["Reranker"]:::model
R --> T
S --> T

T --> U["5.3 Final LLM Response"]:::output
end
```
