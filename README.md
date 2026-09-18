# Proof of Concept - Eagle Trans AI Knowledge Assistant

> An internal AI-powered chatbot for the Germany Export Operations team at Eagle Trans Europe.
> Built as a Proof of Concept during a Summer Internship (June–July 2026).

---

## What Problem Does This Solve?

Every day, before any task begins, someone on the team is searching for information first.

- A customer sends "1x 40HC Hamburg to Mundra" — team needs to check which loading yard and transporter was used last time → opens Outlook, filters Excel
- A new joinee needs to submit VGM → scrolls through SOP PDF page by page or interrupts a senior colleague
- Someone needs free days for Hamburg CTA terminal → searches internal list that does not exist on Google

**The information always existed. Finding it took time.**

This assistant makes all of that searchable in plain English. You type a question. It searches the internal documents. It gives you the answer with the exact source cited — file, page, sheet, row.

---


## How It Works — Simple Flow

```
                    ┌─────────────────────────────────┐
                    │      PHASE 1 — SETUP (Once)     │
                    └─────────────────────────────────┘

  SOP PDF                           Excel Sheets       
     │                                      │
     └──────────────────┼───────────────────┘
                        │
                        ▼
            Split into 512-token chunks
            (50-token overlap at edges)
                        │
                        ▼
         nomic-embed-text converts each chunk
         into 768 numbers (a vector)
                        │
                        ▼
         ChromaDB stores all vectors
         on disk — local laptop only
                        │
                  Knowledge base ready ✓

            ┌─────────────────────────────────┐
            │   PHASE 2 — EVERY QUESTION      │
            └─────────────────────────────────┘

  Employee types: "Which loading yard for Tata Steel?"
                        │
                        ▼
         nomic-embed-text converts question
         into the same 768-number format
                        │
                        ▼
         ChromaDB uses cosine similarity
         to find the top 6 matching chunks
                        │
                        ▼
         Chunks + question sent to
         openai/gpt-oss-20b via Groq API
                        │
                        ▼
         Answer written from document
         content only — never guesses
                        │
                        ▼
         Answer shown in Streamlit browser
         with exact source: file + row/page
```

---

## What It Reads

| Document | Type | What It Contains |
|---|---|---|
| Germany Operations SOP | PDF (9 pages) | Procedures, VGM steps, customs rules, country restrictions |
| Shipment Tracking Excel | Excel — 3 sheets, 100+ rows | Booking records, incharge details, transporter history |

---

## Tech Stack — What Each Tool Does

| Tool | Version | Role | Simple Analogy |
|---|---|---|---|
| Python | 3.11 | Core language — runs everything | The engine |
| LlamaIndex | 0.11.14 | RAG framework — connects all components | The assembly line |
| nomic-embed-text | via Ollama | Converts text to vectors for search | Translates words into numbers computers understand |
| ChromaDB | 0.5.5 | Local vector database | The library catalogue |
| Groq API | latest | Runs the language model in the cloud | The librarian who reads and writes the answer |
| openai/gpt-oss-20b | via Groq | Language model that generates answers | The brain |
| Streamlit | 1.38.0 | Chat interface in the browser | The front desk |
| PyMuPDF | 1.24.9 | Reads PDF files page by page | PDF reader |
| pandas + openpyxl | 3.0+ / 3.1.5 | Reads Excel files row by row | Excel reader |
| python-docx | 1.1.2 | Reads Word documents section by section | Word reader |

---

## RAG Type Used

This project uses **Naive RAG** — the simplest and most appropriate type for this use case.

**Three types of RAG exist:**

| Type | What it does | When to use |
|---|---|---|
| **Naive RAG** ← this project | Load → Chunk → Embed → Retrieve → Answer | Clean, small, structured documents |
| Advanced RAG | Adds query rewriting, re-ranking | Large, messy, diverse document sets |
| Modular RAG | Fully custom pipeline, swappable components | Enterprise production systems |

**Why Naive RAG here:** The Germany SOP is a clean 9-page PDF. The Excel files are structured rows. There is no ambiguity that would require query rewriting or re-ranking. Naive RAG answers correctly and is simple enough to understand, debug, and maintain.

---

## Technical Choices Explained

### Chunking — Fixed Size at 512 tokens, 50 overlap

Each document is split into pieces of 512 tokens. Why 512? It is roughly 400 words — enough to capture a complete SOP procedure step without being so large that irrelevant content gets included in the answer. The 50-token overlap ensures that no important sentence is cut exactly at a boundary.

Alternative considered: Semantic chunking splits based on meaning changes. Rejected because it adds complexity with no real benefit for clean, structured documents like an SOP PDF.

### Embedding — nomic-embed-text

Converts text into 768 numbers representing its meaning. Similar meaning = similar numbers = found together during search. Chosen because it is free, runs fully offline via Ollama, is only 274 MB, and produces good quality embeddings for English operational text.

### Similarity Search — Cosine Similarity (ChromaDB default)

Measures the angle between two vectors to find similar meaning — regardless of text length. Standard choice for semantic text search. Alternative like Euclidean distance measures straight-line distance and works better for numerical data, not text.

### Top K = 6

Retrieves the 6 most similar chunks before generating an answer. Tested with 3 (too narrow, missed context spread across multiple SOP sections) and 8 (added noise, slower). 6 was the best balance for this document set.

### Language Model — openai/gpt-oss-20b via Groq

Groq provides very fast inference (1 to 3 seconds) on open-source models via a free API. The model reads the retrieved chunks and writes a clear, natural language answer based only on that content. It never searches the internet.

---

## Version 1 vs Version 2

| | Version 1 (Ollama — offline) | Version 2 (Groq — faster) |
|---|---|---|
| Language model | Llama 3.2 (3 billion params) | openai/gpt-oss-20b (via Groq) |
| Embedding model | nomic-embed-text | nomic-embed-text (same) |
| Speed | 10 to 20 seconds | 1 to 3 seconds |
| Internet needed | No — 100% offline | Yes — Groq API call |
| Data privacy | Nothing leaves laptop | Query chunks sent to Groq server |
| Cost | Zero — runs on CPU | Free tier (14,400 requests/day) |
| Port | 8501 | 8502 |
| Best for | Real production (sensitive data) | Demo and development |

---

## Project Structure

```
eagle_trans_ai_assistant/
│
├── config.py              ← All settings in one place — change models, paths here
├── requirements.txt       ← All Python packages needed
├── app.py                 ← Streamlit chat interface
│
├── data/
│   ├── sops/              ← Put SOP PDF and DOCX files here
│   ├── excel/             ← Put Excel tracking files here
│   └── outlook_export/    ← Optional: exported email .txt files
│
├── ingestion/
│   ├── __init__.py
│   ├── build_index.py     ← Run this to build the knowledge base
│   ├── load_pdf.py        ← Reads PDF page by page
│   ├── load_excel.py      ← Reads Excel row by row, all sheets
│   ├── load_docx.py       ← Reads Word sections
│   └── load_outlook.py    ← Reads Outlook email exports
│
└── storage/
    └── chroma_db/         ← Knowledge base stored here (created automatically)
```

---

## Setup Instructions (Windows)

### Prerequisites

- Windows 10 or 11
- Python 3.11 — download from python.org (tick "Add to PATH" on install)
- Ollama — download from ollama.com

### Step 1 — Pull the embedding model

Open PowerShell and run:

```bash
ollama pull nomic-embed-text
```

Note: The language model (openai/gpt-oss-20b) runs on Groq cloud — no local download needed for Version 2.

### Step 2 — Clone this repo

```bash
git clone https://github.com/komalvinayak/eagle-trans-ai-assistant.git
cd eagle-trans-ai-assistant
```

### Step 3 — Create virtual environment

```bash
py -3.11 -m venv venv
venv\Scripts\activate
```

You will see `(venv)` at the start of your prompt.

### Step 4 — Install dependencies

```bash
pip install -r requirements.txt
```

Takes 3 to 5 minutes. Wait until the prompt returns.

### Step 5 — Add your Groq API key

Get a free API key from console.groq.com then open `config.py`:

```python
GROQ_API_KEY = "your_groq_api_key_here"
GROQ_LLM_MODEL = "openai/gpt-oss-20b"
```

> **Security note:** Never commit your real API key to GitHub. Add `config.py` to `.gitignore` or use environment variables in production.

### Step 6 — Add documents

Place your files in the correct folders:

```
data/sops/     → GermanySOP.pdf, GermanySOP.docx
data/excel/    → Germany_Dummy_Data.xlsx
```

### Step 7 — Build the knowledge base

```bash
python -m ingestion.build_index
```

You will see:
```
Step 1/4: Gathering documents...
[load_excel] Loaded Germany_Dummy_Data.xlsx - sheets: ['Shipment_Tracking', 'Customer_Preferences', 'Germany_POL_Free_Days']
[load_pdf] Loaded GermanySOP.pdf (9 pages)
Step 4/4: Embedding documents and building the index...
Done. The knowledge base is ready.
```

Run this again any time documents change.

### Step 8 — Launch the app

```bash
streamlit run app.py --server.port 8502
```

Browser opens at `http://localhost:8502`. Make sure Ollama is running in the system tray.

---

## Daily Run Command (Quick Reference)

```bash
D: && cd "ET Summer Internship\Project\Higher Speed\eagle_trans_ai_assistant_v2" && venv\Scripts\activate && streamlit run app.py --server.port 8502
```

---

## Example Questions to Ask

**From the Germany SOP:**
```
Is scrap cargo allowed to UAE?
What documents are required for scrap cargo?
How do I submit VGM in SCOPE?
What is the ZAPP process for Hamburg?
What needs to be done after VGM is submitted?
What is the customs deadline for Germany loading?
```

**From the Excel (Customer and Shipment data):**
```
Which loading yard does Tata Steel prefer?
What is the preferred transporter for Bosch India?
How many loading yards does Tata Steel have?
Free days for Hamburg CTA terminal?
Who is incharge of booking BK2458591?
What loading type does Engro prefer?
```

---

## Business Value

### Why This Matters

| Without the Assistant | With the Assistant |
|---|---|
| Open Outlook, search by customer, filter Excel to find loading yard | Type question, get answer in 2 seconds with source |
| Scroll through 9-page SOP to find VGM steps | Ask "how to submit VGM in SCOPE", get step-by-step answer |
| Ask senior colleague who stops their work | New joinee asks assistant independently |
| Check Google for terminal free days (information is internal only) | Type question, answer from internal list |
| Look up compliance rule for each country per shipment | Ask "is scrap allowed to UAE", get answer from SOP |

### Soft Benefits (More Important Than Time Savings)

- **Knowledge stays in the company** — when an experienced employee leaves, what they know stays accessible
- **Consistent answers** — everyone reads from the same approved SOP, not from memory or personal notebooks
- **Faster onboarding** — new joiners become independent sooner without interrupting senior staff
- **Process compliance** — answers come from approved documents, not from memory or informal advice
- **Scales as team grows** — same architecture works for UK, Canada, US teams with local SOPs
- **Reveals unclear SOPs** — frequently asked questions show which processes are undocumented or unclear

---

## Pros and Cons

### Pros

| What works well | Why |
|---|---|
| Answers are always sourced | Every answer shows exact file, page, sheet, row — fully verifiable |
| Completely offline option (v1) | No data leaves the laptop in Version 1 |
| Free to run | Open-source stack — zero licensing cost |
| Works on small hardware | Laptop CPU is enough for a demo |
| Fast with Groq (v2) | 1 to 3 seconds per answer |
| Easy to add new documents | Drop a file in the data folder, rebuild index |
| Language agnostic | Works on English documents, extensible to other languages |
| No training required | Employees just type questions naturally |

### Cons and Current Limitations

| Limitation | Detail | Possible fix |
|---|---|---|
| Groq sends data externally | Question and retrieved chunks go to Groq's server | Use Version 1 (Ollama) for sensitive data |
| Response time 10 to 20 sec (v1) | Llama 3.2 3b on laptop CPU is slow | GPU server with Llama 70b brings to 2 to 4 sec |
| Dummy data only | No real Eagle Trans customer data was used | Run pilot with real SCOPE export data |
| Single user | Not built for multiple employees simultaneously | Deploy on server, add proper web hosting |
| No SCOPE integration | Reads exported documents, not live SCOPE data | SCOPE API integration in Phase 2 |
| No Outlook live connection | Email exports only, not live Outlook | Microsoft Graph API in production |
| Naive RAG limitations | Does not re-rank or rewrite queries | Upgrade to Advanced RAG for larger document sets |
| Groq model deprecations | Groq deprecates models frequently — model names need updating | Monitor Groq deprecation page, update config.py |
| API key hardcoded in app.py | Security risk if pushed to GitHub | Move to environment variable before production |

---

## Known Issues and Fixes

### Groq model decommissioned error

Groq deprecates models regularly. If you see:
```
model has been decommissioned and is no longer supported
```

Go to `console.groq.com/playground`, open the model dropdown, note which models are listed, and update `config.py`:
```python
GROQ_LLM_MODEL = "openai/gpt-oss-20b"   # or whatever is currently active
```

### API key security

The current `app.py` has the API key hardcoded. Before pushing to GitHub, move it to an environment variable:

```python
import os
GROQ_API_KEY = os.environ.get("GROQ_API_KEY", "")
```

Then set it in your terminal before running:
```bash
set GROQ_API_KEY=gsk_your_key_here
```

---

## Future Scope

| Phase | What | How |
|---|---|---|
| Phase 2 | Connect to real SCOPE export data | SCOPE export CSV loaded nightly via scheduled task |
| Phase 2 | Live Outlook email integration | Microsoft Graph API with IT approval |
| Phase 3 | Deploy for full Germany team | GPU server inside office network, Nginx, HTTPS |
| Phase 3 | European teams (Belgium, Netherlands) | Same code, different knowledge base per region |
| Phase 4 | UK / Canada / US offices | Country-specific knowledge bases, enterprise Azure AD login |
| Future | Upgrade to Advanced RAG | Add query rewriting and re-ranking for larger document sets |
| Future | WhatsApp or Teams integration | Wrap query engine as API, connect to Teams bot |
| Future | Hindi / multilingual support | Multilingual embedding model (e.g. multilingual-e5) |

---

## Production Deployment (On-Premise)

For real company deployment where data must never leave the company:

1. Buy one server with Nvidia RTX 4090 GPU — approximately Rs 1,75,000 one-time cost
2. Install Ollama and pull `llama3.1:8b` (fast, fits in 24GB VRAM)
3. Run Streamlit with `--server.address 0.0.0.0` so the whole network can access it
4. Use Nginx or Cloudflare Tunnel to give it a proper internal URL
5. All documents, all embeddings, all queries — everything stays inside the company network

Monthly cost after that: electricity only — approximately Rs 2,000 to 3,000.

---

## Built By

**Komal Rani**
MBA Business Analytics — Akal University, Talwandi Sabo
Summer Intern — Europe Export Operations
Xenage Solutions Pvt. Ltd. (Eagle Inbrit Group), Pune
June – July 2026

GitHub: [komalvinayak](https://github.com/komalvinayak)

---

## License

Built during an internship at Xenage Solutions Pvt. Ltd. (Eagle Inbrit Group). Shared for educational purposes. Internal company documents, SOPs, and real data are not included in this repository.

---

## Security Reminder

Before pushing to GitHub:
- Remove or rotate the Groq API key visible in `app.py` and `config.py`
- Add `config.py` to `.gitignore`
- Use environment variables for all API keys in production
