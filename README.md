# 🖊️ InkGraph

> **A Neural Blog Generating Agent powered by LangGraph**

InkGraph is an **end-to-end autonomous AI agent** that generates publication-ready technical blog posts on any topic — with no manual writing required. You give it a topic and a date, and it autonomously plans, researches, writes, illustrates, and exports a complete blog article complete with diagrams, citations, and structured markdown.

It is not a simple "prompt → text" pipeline. It is a **stateful multi-agent graph** where specialized AI nodes collaborate to produce high-quality content — just like a real editorial team.

---

## ✨ Features

### 🗺️ 1. Intelligent Topic Routing
- A **Router Agent** analyzes the topic and decides the research strategy before writing anything.
- Three adaptive modes:

  | Mode | When Used | Research |
  |---|---|---|
  | `closed_book` | Evergreen technical concepts | ❌ No web search |
  | `hybrid` | Mix of concepts + current tools | ✅ Selective search |
  | `open_book` | Weekly news, latest AI models, pricing | ✅ Deep web search |

- Automatically adjusts **recency windows** (7 days for news, 45 days for hybrid, evergreen for concepts).

### 🔎 2. Autonomous Web Research (Tavily)
- Fires **up to 10 parallel queries** to Tavily Search (6 results each).
- A **Research Synthesizer LLM** deduplicates by URL, normalizes dates, and filters stale evidence.
- Evidence is passed downstream to ensure **verified citations**, not hallucinations.

### 🗂️ 3. Structured Blog Planning (Orchestrator)
- Produces a full typed `Plan` with blog title, audience, tone, and kind (`explainer`, `tutorial`, `news_roundup`, `comparison`, `system_design`).
- **5–9 section tasks**, each with a goal, 3–6 bullets, target word count, and flags (`requires_research`, `requires_citations`, `requires_code`).

### ⚡ 4. Parallel Section Writing (Worker Fan-out)
- LangGraph's `Send` primitive **fans out all tasks in parallel** to individual Worker agents.
- Each Worker covers all bullets in order, respects word count targets (±15%), and cites only provided evidence URLs.
- Sections are **collected and merged in task ID order** without race conditions.

### 🖼️ 5. AI-Powered Image Generation
- An **Image Planner LLM** identifies up to 3 locations where visuals improve understanding.
- Inserts `[[IMAGE_1]]`, `[[IMAGE_2]]`, `[[IMAGE_3]]` placeholders with descriptive generation prompts.
- Images generated via **FLUX.1-schnell** (Black Forest Labs) through the Hugging Face Inference API and embedded directly into the markdown.

### 🛡️ 6. Graceful Fallback on Image Failures
- If image generation fails (rate limits, quota, network issues), the agent **does not crash**.
- A descriptive fallback block is injected — the blog remains fully readable and exportable no matter what.

### 📝 7. Markdown Export & File Management
- Final blog saved as a clean `.md` file named after the blog title.
- Download as: raw Markdown file, or a full **ZIP bundle** (markdown + all images).

### 📚 8. Past Blog Library
- All previously generated `.md` files listed in the sidebar.
- One-click load into the preview viewer with titles auto-extracted from `# Heading`.

### 📡 9. Live Agent Telemetry Dashboard
- Real-time streaming panel showing: current node, mode, research status, queries fired, evidence count, plan tasks, and sections written.
- Full event log available in the **Logs tab**.

### 📊 10. Structured Evidence & Plan Tabs
- **Evidence Tab**: Research sources in a clean table (title, URL, source, publish date).
- **Plan Tab**: Full structured plan with all section tasks in a dataframe + expandable JSON detail.

---

## 🌟 What Makes InkGraph Unique?

| Aspect | Most AI Writing Tools | InkGraph |
|---|---|---|
| Architecture | Single prompt → response | Multi-node **stateful agent graph** |
| Research | Hallucinated facts | Verified real-time web sources |
| Parallelism | Sequential writing | **Fan-out**: all sections written simultaneously |
| Images | Stock photos / none | **AI-generated diagrams** tailored per section |
| Export | Copy-paste | Full **ZIP bundle** (MD + images) |
| Routing | Fixed pipeline | **Adaptive routing** (3 research modes) |
| Recency | Stale knowledge | Configurable **recency windows** |
| UI | Plain chat box | Premium **glassmorphic dashboard** with live telemetry |
| Resilience | Crashes on API errors | **Graceful fallback** at every failure point |
| Output Format | Raw text | Structured Markdown ready for publication |

---

## ⚙️ System Architecture

```
User Input (Topic + Date)
        │
        ▼
  ┌─────────────┐
  │   Router    │  ← LLM decides: closed_book / hybrid / open_book
  └──────┬──────┘
         │
    (if research needed)
         ▼
  ┌─────────────┐
  │  Research   │  ← Tavily multi-query → LLM synthesizer → EvidencePack
  └──────┬──────┘
         │
         ▼
  ┌──────────────┐
  │ Orchestrator │  ← LLM produces structured Plan (5–9 tasks)
  └──────┬───────┘
         │
    LangGraph Send (fan-out)
    ┌────┴────┬────────┐
    ▼         ▼        ▼
 Worker    Worker   Worker   ← All sections written in parallel
    └────┬────┴────────┘
         │  (sections collected, sorted by task ID)
         ▼
  ┌───────────────┐
  │ merge_content │  ← Assembles full ordered markdown
  └──────┬────────┘
         ▼
  ┌───────────────┐
  │ decide_images │  ← LLM inserts [[IMAGE_N]] placeholders + prompts
  └──────┬────────┘
         ▼
  ┌─────────────────────────┐
  │ generate_and_place_imgs │  ← FLUX.1-schnell → PNG → embedded in MD
  └──────┬──────────────────┘
         ▼
    final .md file + images/ folder
```

### Agent Nodes

| Node | Role |
|---|---|
| **Router** | Decides research mode: `closed_book` / `hybrid` / `open_book` |
| **Research** | Tavily multi-query → LLM synthesizer → structured `EvidenceItem` objects |
| **Orchestrator** | Produces typed `Plan` — title, audience, tone, 5–9 tasks |
| **Workers** | Each task runs in parallel; writes one markdown section with code + citations |
| **merge_content** | Sorts and joins all worker sections into a single markdown document |
| **decide_images** | LLM decides where images add value; inserts `[[IMAGE_N]]` placeholders |
| **generate_and_place_imgs** | Calls HuggingFace FLUX model; embeds PNG images into markdown |

---

## 🚀 Getting Started

### 1. Clone & set up environment

```bash
git clone "https://github.com/Raman7072/Blog_writing_agent"
cd "blog writing agent"

python -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure API keys

Create a `.env` file in the project root:

```env
GROQ_API_KEY=gsk_...
TAVILY_API_KEY=tvly-...          # Optional — enables web research
HF_TOKEN=hf_...                  # Optional — enables image generation
```

> ⚠️ Never commit your `.env` file. It is already listed in `.gitignore`.

### 3. Run the app

```bash
streamlit run frontend.py
```

Open [http://localhost:8501](http://localhost:8501) in your browser.

---

## 🖥️ UI Overview

The Streamlit frontend provides:

- **Sidebar** — Enter a topic, set an as-of date, and click Generate Blog
- **Past Blogs** — Load any previously generated `.md` file from disk
- **🧩 Plan tab** — View the structured blog plan (title, audience, tone, tasks table)
- **🔎 Evidence tab** — Browse research sources gathered by Tavily
- **📝 Preview tab** — Rendered markdown with images + download buttons
- **🖼️ Images tab** — View all generated images; download as zip
- **🧾 Logs tab** — Full event log for debugging each graph step

---

## 📁 Project Structure

```
blog writing agent/
├── backend.py          # LangGraph agent — all nodes, schemas, graph definition
├── frontend.py         # Streamlit UI (glassmorphic dashboard)
├── requirements.txt    # Python dependencies
├── .env                # API keys (gitignored)
├── .gitignore
├── images/             # Generated images saved here
└── *.md                # Generated blog posts saved here
```

---

## 🔬 Tech Stack

| Layer | Technology |
|---|---|
| **Agent Orchestration** | [LangGraph](https://github.com/langchain-ai/langgraph) — `StateGraph`, `Send`, subgraph composition |
| **LLM** | [Groq](https://groq.com) — `llama-3.3-70b-versatile` (ultra-low latency) |
| **Structured Output** | [Pydantic v2](https://docs.pydantic.dev) + `with_structured_output()` |
| **Web Research** | [Tavily Search API](https://tavily.com) |
| **Image Generation** | [FLUX.1-schnell](https://huggingface.co/black-forest-labs/FLUX.1-schnell) via Hugging Face Inference |
| **Image Processing** | [Pillow](https://python-pillow.org) — PIL Image → PNG bytes |
| **Frontend** | [Streamlit](https://streamlit.io) with custom CSS glassmorphism |
| **State Management** | `TypedDict` + `Annotated[List, operator.add]` fan-out reducers |
| **Export** | `zipfile` + `io.BytesIO` in-memory bundling |
| **Environment** | python-dotenv |

---

## 📐 LangGraph Design Patterns

- **`StateGraph`** — The entire pipeline is a typed state machine with strict input/output contracts per node.
- **`Send` primitive** — Enables true parallel fan-out: each section task runs as an independent graph invocation simultaneously.
- **Subgraph composition** — The reducer (merge → decide → generate) is compiled as a reusable `StateGraph` embedded in the main graph.
- **Annotated reducers** — `sections: Annotated[List[tuple], operator.add]` collects parallel worker outputs using addition semantics — no locking needed.
- **Conditional edges** — The router's decision dynamically selects the next node at runtime.
- **Typed Pydantic schemas** — Every LLM call uses `with_structured_output(PydanticModel)` ensuring outputs are valid and type-safe.

---

## 🔑 API Keys

| Key | Required | Purpose |
|---|---|---|
| `GROQ_API_KEY` | ✅ Yes | LLM for all text generation |
| `TAVILY_API_KEY` | ⚡ Optional | Web research (`hybrid` / `open_book` mode) |
| `HF_TOKEN` | 🖼️ Optional | AI image generation via Hugging Face |

> Without Tavily, the agent runs in `closed_book` mode (evergreen topics only).  
> Without `HF_TOKEN`, image generation is skipped and fallback blocks are inserted.

---

## 📤 Output

Each generated blog is saved as a `.md` file in the working directory (e.g. `intro_to_transformers.md`).  
Generated images are saved under `images/`.

Download directly from the **Preview tab**:
- `⬇️ Download Markdown` — the `.md` file
- `📦 Download Bundle (MD + images)` — full ZIP archive

---

## 💡 Why InkGraph?

1. **Content demand is exploding** — Automates the full blog workflow: research → plan → write → illustrate → export.
2. **No hallucinations** — Evidence-grounded writing with verified source citations.
3. **Visual communication** — AI-generated diagrams embedded alongside every relevant section.
4. **True parallelism** — All sections written simultaneously, not one after another.
5. **News-aware** — `open_book` mode with 7-day recency windows for AI roundups and trend articles.
6. **Modular & swappable** — LLM, image model, and search API are all independently replaceable.

---

## 🌐 Deployment

### Streamlit Community Cloud (Recommended — Free)

1. Push to a public GitHub repo
2. Go to [share.streamlit.io](https://share.streamlit.io) → New App
3. Set `frontend.py` as the entry point
4. Add your API keys under **Settings → Secrets**:
   ```toml
   GROQ_API_KEY = "gsk_..."
   TAVILY_API_KEY = "tvly-..."
   HF_TOKEN = "hf_..."
   ```

### Railway / Render

Add a `Procfile`:
```
web: streamlit run frontend.py --server.port $PORT --server.address 0.0.0.0
```

---

## 📄 License

MIT

---

<div align="center">
  <sub>Built with ❤️ using LangGraph · Groq · Tavily · FLUX · Streamlit</sub>
</div>
