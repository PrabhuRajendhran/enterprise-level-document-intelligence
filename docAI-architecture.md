-----

# Intelligent Document Processing (IDP): The "Fail Up" Architecture

**Version:** 2025 Production Standard
**Philosophy:** Minimize compute costs by starting with lightweight tools and only escalating to heavy AI models when necessary.

## 📋 The Tier List Strategy

The system processes documents using a waterfall approach. If a lower tier fails (low confidence or complexity detected), the document escalates to the next tier.

### Tier 1: The "Digital Native" Approach (Fastest)

  * **Best For:** Computer-generated PDFs (contracts, reports, e-books).
  * **Primary Tool:** `pymupdf4llm` (Standard).
  * **Why:** Unlike raw extraction, this library converts PDFs directly into **Markdown**, which is the native language of LLMs. It preserves headers, bold text, and simple tables automatically.
  * **Alternative:** `Marker` (Deep learning based equation/layout formatting).
  * **Cost:** Free.

### Tier 2: The "Standard OCR" Approach (Offline Baseline)

  * **Best For:** Clean scans (300 DPI+) with standard fonts.
  * **Primary Tool:** `OCRmyPDF` (Wrapper around Tesseract).
  * **Why:** It handles deskewing, rotation, and noise cleaning automatically before applying Tesseract, effectively "fixing" the image before reading it.
  * **Alternative:** `PaddleOCR` or `EasyOCR` (Better for non-English or stylized fonts).
  * **Cost:** Free (CPU/GPU).

### Tier 3: The "Layout-Aware" Approach (Tables & Forms)

  * **Best For:** Invoices, Financial Statements, Complex Layouts.
  * **Primary Tool (API):** `Mistral-OCR` or `LlamaParse`.
      * *Note:* LlamaParse is specifically optimized for RAG (Retrieval Augmented Generation).
  * **Primary Tool (Open Source):** `Surya` or `Docling`.
      * *Note:* `Surya` provides state-of-the-art line-level text detection and table extraction.
  * **Cost:** Low (API) to Free (Self-hosted).

### Tier 4: The "VLM" Approach (The Nuclear Option)

  * **Best For:** Handwritten notes, low-contrast receipts, blurry images, and "reasoning" tasks.
  * **Primary Tool (Open Source):** `Qwen2.5-VL`.
      * *Status:* The current price/performance leader for open-source vision.
  * **Primary Tool (Cloud):** `Claude 3.5 Sonnet` or `GPT-4o`.
      * *Status:* Sonnet currently holds the edge for messy handwriting and code snippets.
  * **Cost:** Moderate to High.

-----

## 🧠 The "Smart Router" Logic

To avoid overkill, use a Python script to check document complexity/quality metrics before routing.

### Router Algorithm

1.  **Check Metadata:** Is it a digital PDF? $\rightarrow$ **Tier 1**.
2.  **Check Quality:** Is the image blurry (Laplacian Variance)? $\rightarrow$ **Tier 4**.
3.  **Check Layout:** Does it have complex tables/forms? $\rightarrow$ **Tier 3**.
4.  **Fallback:** If clean scan $\rightarrow$ **Tier 2**.

### Implementation Logic (Python)

```python
import fitz  # PyMuPDF
import cv2
import numpy as np

def smart_route(pdf_path):
    doc = fitz.open(pdf_path)
    page = doc[0]
    
    # 1. Check Text Layer (Digital Native)
    # If >90% of the page area is covered by text blocks, it's digital.
    text_area = 0
    for block in page.get_text("blocks"):
        r = fitz.Rect(block[:4])
        text_area += r.width * r.height
    
    page_area = page.rect.width * page.rect.height
    if (text_area / page_area) > 0.8: 
        return "Tier 1: pymupdf4llm"

    # 2. Render Page as Image for Quality Check
    pix = page.get_pixmap(dpi=72)
    img = np.frombuffer(pix.samples, dtype=np.uint8).reshape(pix.h, pix.w, pix.n)
    
    # Convert to Gray
    if pix.n == 4: 
        img = cv2.cvtColor(img, cv2.COLOR_RGBA2GRAY) 
    else: 
        img = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)

    # 3. Check Blur (Laplacian Variance)
    # Score < 100 usually indicates significant blur.
    blur_score = cv2.Laplacian(img, cv2.CV_64F).var()
    
    if blur_score < 100:
        return "Tier 4: VLM (Image is blurry)"
        
    # 4. Check Layout Complexity
    # Logic to detect lines/tables would go here.
    
    return "Tier 2: Standard OCR"
```

-----

## 💬 Dual-Path Architecture

The system splits into two paths after the router to handle both specific data extraction and general Q\&A.

### Path 1: The Extraction Path (Structured Data)

  * **Goal:** Get specific fields (Date, Total, Vendor) for the database.
  * **Action:** Send document to Tier 3 or 4 with prompt: *"Extract these 5 fields as JSON."*
  * **Storage:** PostgreSQL or MongoDB.

### Path 2: The Chat Path (Unstructured RAG)

  * **Step A: Chunking**
      * Use **Markdown Header Splitting** or **Late Chunking**. Avoid simple character-count splitting to preserve context.
  * **Step B: Indexing**
      * Store in **Qdrant**, **Weaviate**, or **pgvector**.
  * **Step C: Hybrid Retrieval (Critical Upgrade)**
      * Do not rely on Vector search alone.
      * Implement **Hybrid Search**: `Vector Search` (Semantic) + `BM25` (Keyword Match).
      * *Reasoning:* Vector search often misses exact IDs (e.g., "Invoice \#99281"), while BM25 catches them.
  * **Step D: Reranking**
      * Pass top 10 results into a Reranker model (e.g., `bge-reranker-v2-m3`) to sort by true relevance before sending to the LLM.

-----

## 🚀 Summary of 2025 Optimizations

1.  **Markdown Native:** Moved from `pdfplumber` to `pymupdf4llm` to keep formatting intact for the LLM.
2.  **Open Source VLM:** Adopted `Qwen2.5-VL` as the primary cost-effective vision model.
3.  **Blur Detection:** Added Laplacian Variance check to fail-up bad scans immediately.
4.  **Hybrid Search:** Mandated Vector + Keyword search to handle specific document identifiers.















-------------------------------------------------
This is a strong foundation. To make it production-ready for late 2024/2025 standards, we need to modernize the tool selection (especially for the "Digital Native" and "VLM" tiers) and refine the "Router" logic with concrete quality metrics.

Here is the **Optimized System Architecture**.

### 📋 The 2025 Tier List: "Fail Up" Strategy

#### **Tier 1: The "Digital Native" Approach (Fastest)**

  * **Best For:** Computer-generated PDFs (contracts, reports, e-books).
  * **Primary Tool:** **`pymupdf4llm`** (New standard).
      * *Why?* Unlike raw `pdfplumber`, this library converts PDFs directly into **Markdown**, which is the native language of LLMs. It preserves headers, bold text, and simple tables automatically.
  * **Alternative:** **`Marker`** (by Vik Paruchuri).
      * *Why?* It uses deep learning to format equations and complex layouts into perfect Markdown, though it is slower than PyMuPDF.
  * **Cost:** Free.

#### **Tier 2: The "Standard OCR" Approach (Offline)**

  * **Best For:** Clean scans (300 DPI+) with standard fonts.
  * **Primary Tool:** **`OCRmyPDF`** (wrapper around Tesseract).
      * *Why?* It handles deskewing and cleaning automatically before applying Tesseract, effectively "fixing" the image before reading it.
  * **Alternative:** **`PaddleOCR`** or **`EasyOCR`**.
      * *Why?* They often outperform Tesseract on non-English text or slightly stylized fonts.
  * **Cost:** Free (CPU/GPU).

#### **Tier 3: The "Layout-Aware" Approach (Tables & Forms)**

  * **Best For:** Invoices, Financial Statements, Complex Layouts.
  * **Primary Tool (API):** **`Mistral-OCR`** or **`LlamaParse`**.
      * *Why?* These are purpose-built to turn complex visual documents into structured Markdown or JSON. LlamaParse is specifically optimized for RAG (Retrieval Augmented Generation).
  * **Primary Tool (Open Source):** **`Surya`** (by Vik Paruchuri) or **`Docling`** (IBM).
      * *Why?* `Surya` is a state-of-the-art open-source model for line-level text detection and table extraction that rivals commercial tools.
  * **Cost:** Low (API) to Free (Self-hosted).

#### **Tier 4: The "VLM" Approach (The 'Nuclear' Option)**

  * **Best For:** Handwritten notes, low-contrast receipts, blurry images, "reasoning" tasks.
  * **Primary Tool (Open Source):** **`Qwen2.5-VL`** (Vision-Language).
      * *Why?* As of 2025, this is the price/performance king. It rivals GPT-4o in OCR tasks but can be self-hosted or accessed cheaply via API.
  * **Primary Tool (Cloud):** **`GPT-4o`** or **`Claude 3.5 Sonnet`**.
      * *Why?* Claude 3.5 Sonnet currently holds the edge for correctly transcribing extremely messy handwriting and code snippets from images.
  * **Cost:** Moderate to High.

-----

### 🧠 The "Smart Router" Logic (Python Implementation)

You need a programmatic way to decide "Is this messy?" or "Is this digital?" Here is the logic you can implement in Python:

```python
import fitz  # PyMuPDF
import cv2
import numpy as np

def smart_route(pdf_path):
    doc = fitz.open(pdf_path)
    page = doc[0]
    
    # 1. Check Text Layer (Digital Native)
    # If >90% of the page area is covered by text blocks, it's digital.
    text_area = 0
    for block in page.get_text("blocks"):
        r = fitz.Rect(block[:4])
        text_area += r.width * r.height
    
    page_area = page.rect.width * page.rect.height
    if (text_area / page_area) > 0.8: 
        return "Tier 1: pymupdf4llm"

    # 2. Render Page as Image for Quality Check
    pix = page.get_pixmap(dpi=72)
    img = np.frombuffer(pix.samples, dtype=np.uint8).reshape(pix.h, pix.w, pix.n)
    if pix.n == 4: img = cv2.cvtColor(img, cv2.COLOR_RGBA2GRAY) # Convert to Gray
    else: img = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)

    # 3. Check Blur (Laplacian Variance)
    # Score < 100 usually indicates significant blur.
    blur_score = cv2.Laplacian(img, cv2.CV_64F).var()
    
    if blur_score < 100:
        return "Tier 4: VLM (Image is blurry)"
        
    # 4. Check Layout Complexity (Simple Heuristic)
    # Use OpenCV to count horizontal/vertical lines (tables).
    # If many lines -> Tier 3. Else -> Tier 2.
    # ... (Line detection logic here) ...
    
    return "Tier 2: Standard OCR"
```

-----

### 💬 The Architecture: "Extraction + Hybrid Chat"

Your architecture is good, but for 2025, you must upgrade the **Chat Path** to use **Hybrid Search**. Simple vector search fails on exact matches (like Invoice IDs "INV-99281" or SKU codes).

#### 1\. The Extraction Path (Structured)

  * **Trigger:** User uploads document.
  * **Action:** Smart Router selects Tier -\> JSON is extracted -\> Saved to Postgres.
  * **Example Output:** `{"total": 500.00, "date": "2024-01-01", "vendor": "Acme Corp"}`

#### 2\. The Chat Path (Unstructured RAG)

  * **Step A (Chunking):** Do not just chunk by character count. Use **"Late Chunking"** (if using Jina AI embeddings) or **Markdown Header Splitting**. This keeps related text together.
  * **Step B (Indexing):** Store text in **Qdrant** or **Weaviate**.
  * **Step C (Retrieval - CRITICAL UPGRADE):**
      * Use **Hybrid Search**: Run a Vector Search (semantic meaning) + a **BM25 Search** (Keyword matching).
      * *Why?* If a user asks "Show me invoice 55-A", vector search often misses. BM25 catches the exact keyword.
      * **Rerank:** Pass the top 10 results from Hybrid Search into a "Reranker" model (like `bge-reranker-v2-m3`) to sort them by true relevance before sending to the LLM.

### Summary of Recommended Changes

1.  **Replace `pdfplumber` with `pymupdf4llm`** for Markdown-native extraction.
2.  **Adopt `Qwen2.5-VL`** as your open-source Tier 4 fallback.
3.  **Implement Hybrid Search (Vector + Keyword)** in your chat path to handle specific IDs/Dates.
4.  **Add a Blur Detection Step** (Laplacian Variance) to your router to catch bad scans early.
