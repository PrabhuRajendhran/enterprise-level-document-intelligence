Below is (1) the Diagram and (2) the Complete Detailed Outline Plan in the same style and depth as the frameworks you shared — with explicit routing logic, WHY each step matters, and WHAT risks happen if you skip it.


---

✅ (1) End-to-End Architecture Diagram (ASCII)

Full Document Intelligence Pipeline (Path 1 + Path 2)

┌─────────────────────────────────────────┐
                           │                USER QUERY               │
                           └─────────────────────────────────────────┘
                                           │
                                           ▼
╔═══════════════════════════════════════════════════════════════════════╗
║                         PATH 1 — EXTRACTION PIPELINE                  ║
╚═══════════════════════════════════════════════════════════════════════╝

          ┌─────────────────────────────────────────────────────────┐
          │ TIER 0 — INGESTION LAYER (UNPACKERS)                    │
          │ Purpose: Expand containers (email, archives)            │
          └─────────────────────────────────────────────────────────┘
                     │ Body Text            │ Attachments
                     ▼                      ▼
          ┌──────────────────────────────────────────┐
          │ TIER 1 — UNIVERSAL CONVERSION            │
          │ MarkItDown / pymupdf4llm → Markdown      │
          └──────────────────────────────────────────┘
                 │ Text Output
                 ▼
      ┌──────────────────────────────────────────┐
      │ TIER 1.5 — DATA AGENT (CODE INTERPRETER)│
      │ Excel/CSV → DataFrame                    │
      └──────────────────────────────────────────┘
                 │ Clean Text / DataFrames
                 ▼
          ┌──────────────────────────────────────────┐
          │ TIER 2 — STANDARD OCR                    │
          │ OCRmyPDF / PaddleOCR                     │
          └──────────────────────────────────────────┘
                 │
                 ▼
          ┌──────────────────────────────────────────┐
          │ TIER 3 — LAYOUT-AWARE OCR                │
          │ Surya / Mistral-OCR                      │
          └──────────────────────────────────────────┘
                 │
                 ▼
          ┌──────────────────────────────────────────┐
          │ TIER 4 — VLM FAILOVER                    │
          │ Qwen2.5-VL / Claude 3.5                  │
          └──────────────────────────────────────────┘
                 │
                 ▼
      ┌─────────────────────────────────────────────────────┐
      │ FINAL EXTRACTION MODEL (NuMarkdown / GoLLIE / Qwen) │
      │ Purpose: Convert text → Structured Fields (JSON)    │
      └─────────────────────────────────────────────────────┘
                 │
                 ▼
╔═══════════════════════════════════════════════════════════════════════╗
║                     Store Chunks + Metadata in Vector DB              ║
╚═══════════════════════════════════════════════════════════════════════╝

                 │  (chunks + structured fields as metadata)
                 ▼

╔═══════════════════════════════════════════════════════════════════════╗
║                           PATH 2 — CHAT PATH                           ║
╚═══════════════════════════════════════════════════════════════════════╝

      ┌──────────────────────────────────────────────────────────┐
      │ STEP A — INTELLIGENT CHUNKING                            │
      │ Markdown Header / Jina Late Chunking                     │
      └──────────────────────────────────────────────────────────┘
                 │
                 ▼
      ┌──────────────────────────────────────────────────────────┐
      │ STEP B — RETRIEVAL STRATEGY (Fail-Up Waterfall)          │
      │  B1: Hybrid Search (Vector + BM25)                       │
      │  B2: Metadata Filtering (Sweet Spot)                     │
      │  B3: GraphRAG (Nuclear Option)                           │
      └──────────────────────────────────────────────────────────┘
                 │
                 ▼
      ┌──────────────────────────────────────────────────────────┐
      │ STEP C — RERANKING                                       │
      │ bge-reranker-v2-m3                                       │
      └──────────────────────────────────────────────────────────┘
                 │
                 ▼
         ┌─────────────────────────────────────────┐
         │ FINAL LLM ANSWERING LAYER               │
         │ GPT-5 / Claude / Qwen 72B / Llama 70B   │
         └─────────────────────────────────────────┘
                 │
                 ▼
        ┌──────────────────────────────────────────┐
        │                ANSWER TO USER            │
        └──────────────────────────────────────────┘


---

✅ (2) Detailed Outline Plan (Professional, Exhaustive, Actionable)

This mirrors your style: what each step does, why it matters, what breaks if skipped, and routing logic.


---

🔷 PATH 1 — EXTRACTION PIPELINE (Document → Text → Structured Data)

Goal: Produce clean text and structured fields that can power Path 2 retrieval.


---

TIER 0 — INGESTION LAYER (UNPACKERS)

Best For: Emails (.msg/.eml/.pst), archives (.zip)
Tools: extract-msg, libratom, zipfile

What It Does

Extracts email body

Extracts attachments

Normalizes formats (.pst → .eml)

Recursively unpacks nested files


Why It Matters

You cannot vectorize or extract anything inside a container.
If you skip this, your system never sees:

Attachments

Inline PDFs

Forwarded chains

Embedded documents


Routing Logic

If file is container → Tier 0
Else → Tier 1


---

TIER 1 — UNIVERSAL CONVERSION (Native Digital → Markdown)

Tools: MarkItDown, pymupdf4llm

What It Does

Converts Word, PowerPoint, HTML, XML, digital PDFs into clean Markdown

Preserves headers, tables, lists

Produces highly chunkable structure


Why It Matters

If you skip clean Markdown conversion:

Chunking becomes random

Retrieval quality drops 40–70%

Headers get lost → GraphRAG is useless

Table relationships disappear


Routing Logic

If native digital (docx, pptx, html) → MarkItDown
If digital PDF → pymupdf4llm
Else → Tier 2


---

TIER 1.5 — DATA AGENT (Tabular Intelligence)

Tools: Pandas, PandasAI, LlamaIndex PandasQueryEngine

What It Does

Loads Excel/CSV → DataFrame

Allows LLM-generated code to answer numeric queries


Why It Matters

LLMs CANNOT reason over raw CSV text.
If skipped:

"Total spend in Jan?" → hallucination

"Which vendor had highest cost?" → wrong answer

RAG completely fails on numbers


Routing Logic

If file type ∈ [xlsx, csv, parquet] → Data Agent
Else → T2


---

TIER 2 — STANDARD OCR (Baseline)

Tools: OCRmyPDF, PaddleOCR

What It Does

Extracts text from clean scans (~300 DPI)

Removes noise

Produces readable plain text


Why It Matters

If skipped:

Clean scans become invisible

System thinks file is empty

All retrieval fails downstream


Routing Logic

If scanned PDF & clear text → Tier 2
Else → Tier 3


---

TIER 3 — LAYOUT-AWARE OCR (For Forms / Invoices)

Tools: Surya, Mistral-OCR

What It Does

Detects bounding boxes

Maintains row/column alignment

Handles invoice tables, tax forms, statements


Why It Matters

If skipped:

Tables become garbled text

Line items mix together

Totals, subtotals, taxes become indistinguishable


Routing Logic

If structured layout (invoice/forms) → Tier 3
Else → Tier 4


---

TIER 4 — VLM FAILOVER (The Nuclear OCR)

Tools: Qwen2.5-VL, Claude 3.5 Sonnet, GPT-5 Vision

What It Does

Handles handwriting

Blurry images

Irregular layout

Reasoning-based OCR


Why It Matters

If skipped:

Any imperfect photo becomes unreadable

Edge-case documents fail silently


Routing Logic

If OCR fails or confidence < threshold → Tier 4
Else → Final Extraction


---

FINAL EXTRACTION MODEL (NuMarkdown / GoLLIE / Qwen / Llama)

What It Does:
Turns text → structured fields such as:

{
  "invoice_number": "9921",
  "vendor": "Acme",
  "total": 511.24,
  "date": "2024-10-11"
}

Why It Matters

Skipping structured extraction:

No metadata

Path 2 retrieval becomes blind

Cannot do metadata filtering

Cannot do relationship-based queries


This is the engine that powers Path 2.


---

🔷 PATH 2 — CHAT PATH (Advanced Retrieval)

Goal: Turn user queries → correct document answers.


---

STEP A — INTELLIGENT CHUNKING

Tools: LangChain Markdown Header Splitting, Jina AI Late Chunking

What It Does

Breaks documents along semantic boundaries

Preserves heading hierarchy

Improves context integrity


Why It Matters

If skipped:

Answers come from random fragments

Incorrect page contexts

GraphRAG cannot form nodes

Reranking becomes ineffective



---

STEP B — RETRIEVAL STRATEGY (Fail-Up Cascade)

B1 — Hybrid Search (Vector + BM25)

Vector → semantic match

BM25 → keyword precision


If skipped:

Exact ID/code lookups fail

“Find invoice 9921” → fails with pure embeddings



---

B2 — Metadata Filtering (Sweet Spot)

Uses extracted fields from Path 1.

vendor

total

date

file type

confidence

language


Why It Matters
This solves 95% of business queries.
If skipped:

“Invoices from Acme > $500” is impossible

Retrieval becomes slow + inaccurate



---

B3 — GraphRAG (Nuclear Option)

Best For: Multi-hop questions

Example:
“Who managed the project that generated the highest cost invoice?”

Why It Matters: If skipped, all cross-document reasoning fails.

Routing Logic:

If question references relationships between entities → GraphRAG
Else → B2


---

STEP C — RERANKING (Quality Filter)

Model: bge-reranker-v2-m3
Role: Reorder top 20 results by relevance.

If skipped:

Irrelevant chunks get surfaced

LLM hallucinates due to bad context



---

FINAL ANSWERING LAYER

LLM: GPT-5, Claude 3.7, Qwen2.5-72B, Llama-3.2-90B

Takes reranked chunks → produces final answer.

If skipped:

No coherent summary

No reasoning

No natural language



---

🔷 Routing Intelligence (FULL LOGIC TREE)

1. Input File Routing

if container → Tier 0
elif native digital → Tier 1
elif spreadsheet → Tier 1.5
elif clean scan → Tier 2
elif structured scan → Tier 3
else → Tier 4

2. Extraction Model Routing

If fields are simple → GoLLIE or NuMarkdown-8B
If ambiguous/messy → NuMarkdown-8B-Thinking
If multilingual-heavy → Qwen2.5
If long-form reasoning → Llama-3.1 or Qwen2.5-14B

3. Retrieval Routing

If question is simple lookup → Hybrid Search
If question references metadata → Metadata Filtering
If question is multi-hop or cross-document → GraphRAG

4. Answer Composition

If <5 chunks → direct LLM answer
Else → rerank + fuse + answer


---

🎁 If you want next:

I can produce any of the following:

✅ A PDF architecture document
✅ A Mermaid diagram version
✅ A fully-packaged system design document
✅ A full project plan with tools, code stacks, and infra decisions
✅ Or generate Python code templates for every tier

Just tell me what format you want.
