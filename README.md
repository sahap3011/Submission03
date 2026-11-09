# Grounded Q&A: A Cited Document Assistant

> **Mini Project 3 (Week 12 Deliverable)**  
> A trustworthy Retrieval-Augmented Generation (RAG) assistant over a Sherlock Holmes short-story corpus with inline source citations, citation validation, score-gated abstention, and an automated KPI evaluation harness.

---

## 📌 Overview

This project extends the MP2 Mini-RAG pipeline (`load → chunk → embed → store → retrieve → answer`) with the three capabilities that separate a demo from a production-trustworthy assistant:

1. **Inline Citations & Validation (`format_context`, `validate_citations`)**: Every retrieved chunk is numbered `[1..n]` in the prompt context, every claim in the answer is tagged inline (`[1]`, `[2]`), and all `[n]` markers are parsed and validated against retrieved sources to detect fabricated citations.
2. **Two-Layer Abstention (`should_abstain` + Prompt Guard)**: When no chunk meets the cosine similarity cutoff (`RETRIEVAL_THRESHOLD = 0.49`) or the retrieved excerpts lack the answer, the assistant replies with `"I don't know based on the provided documents."` rather than hallucinating from pre-training memory.
3. **KPI Evaluation Harness (`evaluate`)**: Measures **Retrieval Hit Rate**, **Grounded & Cited Response Rate**, **Abstention Accuracy**, and **Hallucination Rate** over a 9-question evaluation suite (6 answerable + 3 should-abstain).

### Architecture & Tech Stack

```
corpus/*.txt  ──> Paragraph & Header Chunking  ──>  text-embedding-3-small (1536d)
                                                            │
                                                            ▼
                                                      Qdrant Store
                                                            │
User Question ──> Query Embedding ──> Top-k Search ─────────┘
                                            │
                                            ▼
                               Score Gate: should_abstain()
                                  ├── max(score) < threshold ──> "I don't know based on the provided documents."
                                  └── max(score) >= threshold
                                            │
                                            ▼
                                   format_context([1..k])
                                            │
                                            ▼
                                gpt-4o-mini (temperature=0)
                                            │
                                            ▼
                                  validate_citations([n])
                                            │
                                            ▼
                              Cited Answer + Validated Sources
```

- **LLM**: OpenAI `gpt-4o-mini` (`temperature=0`)
- **Embeddings**: OpenAI `text-embedding-3-small` (`1536` dimensions, cosine distance)
- **Vector Database**: Qdrant (Supports Qdrant Cloud, local Docker, or zero-setup local disk persistence `qdrant_storage/`)
- **Corpus**: 5 public-domain Sherlock Holmes story summaries (`corpus/*.txt`, 73 chunks)

---

## 📁 Repository Structure

```
.
├── .gitignore
├── README.md                                  # Root overview and execution guide
├── Week 12_Graded Mini Project.docx           # Assignment guidelines & requirements
└── AI-RAG_MP3_GroundedQA/
    ├── AI-RAG_MP3_Brief.md                    # Project brief and instructions
    ├── README.md                              # Bundle specifications
    ├── requirements.txt                       # Python dependencies
    └── learner/
        ├── .env.example                       # Environment template
        ├── requirements.txt                   # Dependencies
        ├── MP3_FAQ.md                         # Frequently asked questions
        ├── mp3_grounded_qa.py                 # Complete Grounded Q&A assistant (4 functions implemented)
        ├── mp3_grounded_qa_starter.py         # Original starter skeleton
        ├── mp3_eval.txt                       # Evaluation table output (main graded artifact)
        ├── mp3_reflection.md                  # Reflection report + Stretch threshold sweep
        ├── mp3_demo_guide.md                  # 2–3 minute demo walkthrough & live transcript
        ├── corpus/                            # 5 Sherlock Holmes story summaries
        │   ├── 01_red_headed_league.txt
        │   ├── 02_speckled_band.txt
        │   ├── 03_blue_carbuncle.txt
        │   ├── 04_engineers_thumb.txt
        │   └── 05_scandal_in_bohemia.txt
        └── data/
            ├── predefined_questions.jsonl     # 6 predefined questions (4 answerable + 2 abstain)
            └── learner_questions.jsonl        # 3 learner-authored questions (2 answerable + 1 abstain)
```

---

## 🚀 Setup & Installation

```bash
cd "AI-RAG_MP3_GroundedQA/learner"
python -m venv .venv
# Windows PowerShell: .venv\Scripts\Activate.ps1
# macOS / Linux:      source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # Set OPENAI_API_KEY and QDRANT_URL / QDRANT_PATH
```

---

## ⚡ Running the Project

### 1. Offline Self-Test (`selftest` — no API keys needed)
```bash
python mp3_grounded_qa.py selftest
```

### 2. Ingest Corpus (`ingest`)
```bash
python mp3_grounded_qa.py ingest
```

### 3. Interactive Grounded Q&A (`ask`)
```bash
python mp3_grounded_qa.py ask
```

### 4. Evaluation Harness (`eval`)
```bash
python mp3_grounded_qa.py eval
```

**Evaluation Output (`mp3_eval.txt`):**
```text
  id                              type     hit   abstained  grounded  ok
  --------------------------------------------------------------------------
  q1_red_headed_league_assistant  answer   ✓     ✓          ✗         ✗
  q2_speckled_band_weapon         answer   ✓     ✓          ✗         ✗
  q3_scandal_bohemia_adler        answer   ✓     ✗          ✓         ✓
  q4_engineers_thumb_press        answer   ✓     ✗          ✓         ✓
  abstain_moriarty                abstain  -     ✓          -         ✓
  abstain_out_of_domain           abstain  -     ✓          -         ✓
  q5_blue_carbuncle_theft         answer   ✓     ✗          ✓         ✓
  q6_speckled_band_multihop       answer   ✓     ✗          ✓         ✓
  q7_abstain_mycroft_diogenes     abstain  -     ✓          -         ✓
  --------------------------------------------------------------------------

  Retrieval hit rate    : 1.0   (6 answerable questions)
  Grounded & cited rate : 0.667   (KPI: grounded response rate)
  Abstention accuracy   : 1.0   (3 should-abstain questions)
  Hallucination rate    : 0.0   (KPI: hallucination rate)
```

---

## 📑 Deliverables Checklist

- [x] **`mp3_grounded_qa.py`**: Assistant with `format_context`, `validate_citations`, `should_abstain`, and `evaluate` implemented and verified via `selftest`.
- [x] **`mp3_eval.txt`**: Output of `python mp3_grounded_qa.py eval`.
- [x] **`data/learner_questions.jsonl`**: 3 custom questions (`q5_blue_carbuncle_theft`, `q6_speckled_band_multihop`, `q7_abstain_mycroft_diogenes`).
- [x] **`mp3_reflection.md`**: Reflection covering grounding & abstention, citations, eval table analysis, one surprise, and the stretch threshold sweep.
- [x] **`mp3_demo_guide.md`**: Live demo script and transcript showing cited answers and abstention.
