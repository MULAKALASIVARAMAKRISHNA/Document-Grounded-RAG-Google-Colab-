# Document-Grounded RAG (Google Colab)

Ask questions about a document you upload and get answers **only from that document**. Anything the document doesn't contain is refused instead of guessed.

Runs entirely in a Google Colab notebook. There is no UI to set up, just upload a file and ask.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR_USERNAME/YOUR_REPO/blob/main/document_rag_v3.ipynb)

> Replace `YOUR_USERNAME` and `YOUR_REPO` in the badge link after you push this to GitHub.

---

## Features

- **Supports PDF, DOCX, TXT and MD** files.
- **Small documents are sent whole.** Resumes, letters and short reports (up to `MAX_CHARS`, default 24,000 characters) skip retrieval entirely, so short questions like `college name` or `summarise that` work.
- **Hybrid retrieval for larger documents:** BM25 + dense embeddings, merged with Reciprocal Rank Fusion, then re-ranked with a cross-encoder.
- **Relevance gate:** clearly unrelated questions are refused without calling an LLM.
- **Multi-provider LLM chain:** Sarvam first, then Groq, with automatic fallback if a model is down, rate-limited or unavailable.
- **Strict JSON output with auto-repair:** if a model returns invalid JSON, it is asked again with the error.
- **Citations:** every answer lists the excerpts it used, with page numbers and the most relevant snippet.
- **Grounding check:** warns when numbers in an answer don't appear in the cited text.
- **Prompt-injection resistant:** the document and the question are treated as untrusted data.
- **Evaluation harness** to test your own question/answer cases (run in parallel).

## Requirements

- A Google account (for Colab)
- At least one API key:
  - [Sarvam AI](https://www.sarvam.ai/) → `SARVAM_API_KEY`
  - [Groq](https://console.groq.com/) → `GROQ_API_KEY`

## Quick start

1. Open `document_rag_v3.ipynb` in Google Colab (use the badge above, or **File → Upload notebook**).
2. Add your API keys to **Colab Secrets** (the key icon in the left sidebar):
   - Name: `SARVAM_API_KEY` and/or `GROQ_API_KEY`
   - Value: your key
   - Turn on **Notebook access** for each secret.
3. Run the cells from top to bottom.
4. When the upload cell runs, choose your PDF, DOCX, TXT or MD file.
5. Run the question cell, or use the interactive loop to ask your own questions. Type `exit` to stop.

The health-check table printed when the engine starts shows which models are `WORKING`, `RATE-LIMITED` (kept) or `FAILED` (removed from the chain).

## Example

```
Q: What is the name of the person?
A: Jane Doe
   [confidence 1.00 | sarvam:sarvam-105b | 2.8s]
   - [1] page 1: ...Jane Doe M.Sc. Computer Science...

Q: What is the capital of France?
A: The uploaded document does not contain this information.
   [refused | sarvam:sarvam-105b | 2.1s]
```

## Configuration

All settings are in the **SETTINGS** cell near the top of the notebook.

| Setting | Default | What it does |
|---|---|---|
| `PROVIDER_ORDER` | `["sarvam", "groq"]` | Providers tried in order. A provider without a key is skipped. |
| `SARVAM_MODEL` | `sarvam-105b` | Sarvam model name. |
| `GROQ_MODEL_PREFERENCE` | Qwen, then two Llama models | Groq models in priority order. Models your key can't access are skipped automatically. |
| `SARVAM_MAX_TOKENS` | `4000` | Token budget for Sarvam. Reasoning models need room to think, or they return empty output. |
| `SARVAM_REASONING_EFFORT` | `"low"` | Less thinking means faster answers. Dropped automatically if the API rejects it. |
| `MAX_REPAIR_ROUNDS` | `2` | How many times to re-ask a model that returns invalid JSON. |
| `ENABLE_GROUNDING_CHECK` | `True` | Warn when answer numbers aren't found in the cited text. |
| `CONCURRENCY` | `2` | Parallel questions in the evaluation harness only. |
| `MAX_CHARS` | `24000` | Documents up to this size are sent whole. Larger ones use retrieval. |

Retrieval settings such as chunk size, overlap and the relevance threshold are in the `Config` class.

## How it works

```
Upload → extract text → clean
   ├─ small document (≤ MAX_CHARS): send whole text to the LLM
   └─ large document: chunk → BM25 + dense search → RRF fusion
                      → cross-encoder rerank → relevance gate
                                  ↓
        LLM chain (Sarvam → Groq) returns strict JSON:
        {answerable, answer, citations, confidence}
                                  ↓
        verify citations + numbers → show answer with sources
```

Models used for retrieval: `sentence-transformers/all-MiniLM-L6-v2` (embeddings) and `cross-encoder/ms-marco-MiniLM-L-6-v2` (re-ranking). Both download automatically on first run.

Chunks and embeddings are cached in `/content/rag_cache`, so re-uploading the same file is fast.

## Evaluating your own document

In the evaluation cell, add cases in this format:

```python
cases = [
    ("What is the total contract value?", True, ["50,000"]),   # must answer and include "50,000"
    ("What is the capital of France?", False, []),            # must be refused
]
evaluate(rag, cases)
```

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `No usable provider` | No API key was found. Add a key in Colab Secrets and enable notebook access. |
| A model shows `FAILED` in the health check | The model name is wrong or not available for your key. Run the Groq model-list cell, or read the error text shown next to the model. |
| `429` / "Request too large" (Groq) | The request exceeds that model's per-minute token limit on your plan. Lower `MAX_CHARS` (e.g. `12000`) or put a different model first in `GROQ_MODEL_PREFERENCE`. |
| Sarvam returns empty or truncated output | Raise `SARVAM_MAX_TOKENS`. The notebook also retries automatically with more tokens. |
| Short questions are refused with `retrieval-gate` | Only happens with large documents. Lower `min_dense_score` in `Config`, or raise `MAX_CHARS` so the document is sent whole. |
| `No extractable text found` | The PDF is scanned (images only). Run OCR on it first. |
| `All providers failed` | Every model errored. Read the error list in the message, then check keys, model names and rate limits. |

## Limitations

- Scanned PDFs and images need OCR before use.
- One document per session. Upload a new file and re-run the ingest cell to switch.
- Questions are answered independently, with no chat memory, so follow-ups like "summarise that" are treated as questions about the document.
- Tables and multi-column layouts may extract imperfectly from PDFs.
