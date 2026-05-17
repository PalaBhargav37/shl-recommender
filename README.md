# SHL Assessment Recommender

A conversational AI agent that helps hiring managers navigate the SHL assessment catalog through dialogue. Built with FastAPI + Claude (Anthropic API).

---

## What It Does

- Takes a **vague hiring intent** ("I need to hire a Java developer") and guides the user to a **grounded shortlist** of SHL assessments through multi-turn conversation
- **Clarifies** before recommending — never jumps to conclusions on vague queries
- **Refines** recommendations mid-conversation when the user changes constraints
- **Compares** assessments when asked, using only catalog data
- **Stays in scope** — refuses general HR advice, legal questions, and prompt injections
- Every URL returned is verified against the catalog — no hallucinated links

---

## Project Structure

```
shl-recommender/
├── app/
│   ├── __init__.py
│   └── main.py            # FastAPI app — all endpoints, prompt, LLM logic
├── data/
│   ├── __init__.py
│   └── catalog.py         # Full SHL catalog as Python list (no DB needed)
├── tests/
│   ├── __init__.py
│   └── test_agent.py      # Pytest test suite
├── .env.example           # Copy to .env and add your API key
├── .gitignore
├── Procfile               # For Railway / Heroku deployment
├── render.yaml            # For Render.com deployment
├── requirements.txt
└── README.md
```

---

## Prerequisites

| Tool | Version | Download |
|------|---------|----------|
| Python | 3.11 or 3.12 | https://www.python.org/downloads/ |
| VS Code | Latest | https://code.visualstudio.com/ |
| Git | Any recent | https://git-scm.com/ |
| Anthropic API Key | — | https://console.anthropic.com/ |

---

## Step-by-Step Setup in VS Code

### Step 1 — Open the project in VS Code

1. Unzip `shl-recommender.zip` to any folder (e.g., your Desktop)
2. Open **VS Code**
3. Click **File → Open Folder** and select the `shl-recommender` folder
4. VS Code will show the project files in the Explorer on the left

---

### Step 2 — Install the Python extension (first time only)

1. Click the **Extensions** icon in the left sidebar (looks like 4 squares)
2. Search for **Python** (by Microsoft)
3. Click **Install**

---

### Step 3 — Open the integrated terminal

In VS Code: press `` Ctrl+` `` (backtick) — or go to **Terminal → New Terminal**

All commands below are run in this terminal.

---

### Step 4 — Create a virtual environment

```bash
# Windows
python -m venv .venv

# macOS / Linux
python3 -m venv .venv
```

Then **activate** it:

```bash
# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# Windows (Command Prompt)
.venv\Scripts\activate.bat

# macOS / Linux
source .venv/bin/activate
```

You should see `(.venv)` at the start of your terminal prompt.

> **VS Code tip:** Press `Ctrl+Shift+P` → type **Python: Select Interpreter** → choose the `.venv` one. This means VS Code uses your virtual environment for IntelliSense and running code.

---

### Step 5 — Install dependencies

```bash
pip install -r requirements.txt
```

This installs FastAPI, Uvicorn, HTTPX, and Pydantic.

---

### Step 6 — Set your API key

1. In VS Code Explorer, right-click `.env.example` → **Copy**
2. Right-click in the Explorer panel → **Paste** → rename the copy to `.env`
3. Open `.env` and replace `sk-ant-your-key-here` with your real Anthropic API key:

```
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Get your key at: https://console.anthropic.com/settings/api-keys

---

### Step 7 — Run the server

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

You should see:
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started reloader process
```

The `--reload` flag means the server auto-restarts when you save code changes.

---

### Step 8 — Verify it works

Open your browser and go to: **http://localhost:8000/health**

You should see:
```json
{"status": "ok"}
```

Also open: **http://localhost:8000/docs** — this is the interactive Swagger UI where you can test the API in your browser.

---

## Testing the Chat Endpoint

### Option A — Swagger UI (browser, no code)

1. Go to **http://localhost:8000/docs**
2. Click **POST /chat → Try it out**
3. Paste this into the Request Body:

```json
{
  "messages": [
    {"role": "user", "content": "I'm hiring a mid-level Java developer who works with stakeholders."}
  ]
}
```

4. Click **Execute**

---

### Option B — curl (terminal)

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "I need to hire a Java developer, 4 years experience."}
    ]
  }'
```

---

### Option C — VS Code REST Client extension

1. Install the **REST Client** extension in VS Code
2. Create a file `test.http` and paste:

```http
### Health check
GET http://localhost:8000/health

### Vague query (should ask clarifying question)
POST http://localhost:8000/chat
Content-Type: application/json

{
  "messages": [
    {"role": "user", "content": "I need an assessment"}
  ]
}

### Specific query (should return recommendations)
POST http://localhost:8000/chat
Content-Type: application/json

{
  "messages": [
    {"role": "user", "content": "Hiring a mid-level Java developer, 4 years exp, works with stakeholders."}
  ]
}

### Multi-turn conversation
POST http://localhost:8000/chat
Content-Type: application/json

{
  "messages": [
    {"role": "user", "content": "We need a solution for senior leadership."},
    {"role": "assistant", "content": "Happy to help. Who is this meant for - selection, development, or something else?"},
    {"role": "user", "content": "CXO and Director level, for selection against a leadership benchmark."}
  ]
}
```

3. Click **Send Request** above each block

---

## Running the Tests

Make sure your `.env` file has a valid API key, then:

```bash
pytest tests/ -v
```

The test suite checks:
- ✅ `/health` returns `{"status": "ok"}`
- ✅ Schema compliance on every response
- ✅ All recommendation URLs exist in the catalog (no hallucinations)
- ✅ Vague queries don't get immediate recommendations
- ✅ Off-topic and legal questions are refused
- ✅ Prompt injection is refused
- ✅ Max 10 recommendations enforced
- ✅ Refinement honors add/remove requests
- ✅ `end_of_conversation` is not set prematurely
- ✅ JSON parsing handles markdown fences and malformed output

Run a single test:
```bash
pytest tests/test_agent.py::test_health -v
```

Run only unit tests (no API calls):
```bash
pytest tests/test_agent.py::test_parse_valid_json tests/test_agent.py::test_parse_hallucinated_url_stripped tests/test_agent.py::test_parse_strips_markdown_fences -v
```

---

## Example Conversations

### Java Developer Hire

**Request:**
```json
{
  "messages": [
    {"role": "user", "content": "Hiring a Java developer who works with stakeholders"},
    {"role": "assistant", "content": "Sure. What is seniority level?"},
    {"role": "user", "content": "Mid-level, around 4 years"}
  ]
}
```

**Response:**
```json
{
  "reply": "Got it. Here are 5 assessments that fit a mid-level Java dev with stakeholder needs.",
  "recommendations": [
    {"name": "Java 8 (New)", "url": "https://www.shl.com/...", "test_type": "K"},
    {"name": "Core Java (Advanced Level) (New)", "url": "https://www.shl.com/...", "test_type": "K"},
    {"name": "Spring (New)", "url": "https://www.shl.com/...", "test_type": "K"},
    {"name": "SHL Verify Interactive G+", "url": "https://www.shl.com/...", "test_type": "A"},
    {"name": "Occupational Personality Questionnaire OPQ32r", "url": "https://www.shl.com/...", "test_type": "P"}
  ],
  "end_of_conversation": false
}
```

### Vague Query (Clarification Required)

**Request:**
```json
{"messages": [{"role": "user", "content": "I need an assessment"}]}
```

**Response:**
```json
{
  "reply": "Happy to help! To point you to the right assessments, could you tell me what role or function you're hiring for?",
  "recommendations": [],
  "end_of_conversation": false
}
```

---

## Deployment

### Render.com (Free tier — recommended)

1. Push your code to a GitHub repo (make sure `.env` is in `.gitignore`)
2. Go to https://render.com → **New Web Service**
3. Connect your GitHub repo
4. Render detects `render.yaml` automatically
5. In **Environment Variables**, add `ANTHROPIC_API_KEY` = your key
6. Click **Deploy**

Your service will be live at `https://your-service-name.onrender.com`

> **Note:** Free tier services sleep after inactivity. The `/health` endpoint allows up to 2 minutes for cold start (as specified in the assignment).

### Railway.app

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login and deploy
railway login
railway init
railway up

# Set env var
railway variables set ANTHROPIC_API_KEY=sk-ant-xxx
```

### Local with Docker

```dockerfile
# Dockerfile (create this if you want Docker)
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t shl-recommender .
docker run -e ANTHROPIC_API_KEY=sk-ant-xxx -p 8000:8000 shl-recommender
```

---

## Design Decisions

### Why no vector database?
The catalog has ~120 items — small enough to embed entirely in the system prompt as a compact text summary. This eliminates cold-start latency, removes infrastructure dependencies, and keeps the project self-contained. For a 500+ item catalog, a vector store (FAISS/Chroma) would be worthwhile.

### Why Claude as the backbone?
The assignment allows any LLM. Claude was chosen because: (a) it follows structured JSON output instructions reliably, (b) it understands nuanced domain constraints ("refuse legal questions but answer what a test measures"), and (c) it handles multi-turn refinement gracefully without drifting.

### Why validate URLs server-side?
LLMs can hallucinate URLs even when given explicit instructions. The `parse_agent_response` function strips any recommendation whose URL isn't in the catalog, ensuring the hard eval requirement ("items from catalog only") is always met regardless of model behavior.

### Stateless API design
The API is stateless as required — the full conversation history is sent on every POST /chat call. No server-side session storage needed.

### Turn cap
Messages are trimmed to the last 8 if the conversation exceeds the limit, preventing timeouts and context overflow.

---

## API Reference

### `GET /health`
Returns `{"status": "ok"}` with HTTP 200. Used for readiness probes.

### `POST /chat`

**Request body:**
```json
{
  "messages": [
    {"role": "user", "content": "string"},
    {"role": "assistant", "content": "string"}
  ]
}
```

**Response:**
```json
{
  "reply": "string",
  "recommendations": [
    {
      "name": "string",
      "url": "string",
      "test_type": "string"
    }
  ],
  "end_of_conversation": false
}
```

**Test type codes:**
- `A` = Ability & Aptitude
- `B` = Biodata & Situational Judgment
- `C` = Competencies
- `D` = Development & 360
- `E` = Assessment Exercise / Virtual AC
- `K` = Knowledge & Skills
- `P` = Personality & Behavior
- `S` = Simulations

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `ModuleNotFoundError: No module named 'fastapi'` | Run `pip install -r requirements.txt` inside your activated `.venv` |
| `AuthenticationError` from Anthropic | Check your `.env` file has a valid `ANTHROPIC_API_KEY` |
| `python-dotenv` not loading `.env` | Ensure the `.env` file is in the project root (same folder as `requirements.txt`) |
| Port 8000 already in use | Run with `--port 8001` or kill the other process |
| Tests fail on API tests | API tests make real calls — ensure your key has credits and network is available |
| `.venv\Scripts\Activate.ps1 cannot be loaded` (Windows) | Run: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` |

---

## Loading .env Automatically

The server auto-loads `.env` via `python-dotenv`. If you want to be explicit, add this to the top of `app/main.py` (already handled via uvicorn's env loading):

```python
from dotenv import load_dotenv
load_dotenv()
```

Or export the key directly in your terminal session:

```bash
# macOS/Linux
export ANTHROPIC_API_KEY=sk-ant-xxx

# Windows PowerShell
$env:ANTHROPIC_API_KEY="sk-ant-xxx"
```
