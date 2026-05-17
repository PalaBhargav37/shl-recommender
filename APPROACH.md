# Approach Document — SHL Assessment Recommender

## Problem Framing

The task is to convert a hiring manager's vague intent into a grounded shortlist of SHL assessments through multi-turn dialogue. The core challenge is threefold: (1) knowing *when* to ask versus *when* to recommend, (2) ensuring every recommendation is grounded in real catalog data with no hallucinations, and (3) handling the full range of conversational behaviors (clarification, refinement, comparison, refusal) robustly.

---

## Design Choices

### 1. Catalog Strategy — Embedded Summary in Prompt

Rather than a vector store or database, the entire catalog (~120 assessments) is embedded as a compact text summary directly in the system prompt. Each line encodes: `entity_id | name | test_type | keys | duration | job_levels | languages | url`.

**Why:** The catalog fits comfortably within Claude's context window. This eliminates cold-start latency, removes infrastructure dependencies, and makes the service self-contained — critical for a 30-second timeout constraint. For a catalog of 500+ items, a FAISS or Chroma vector store with semantic retrieval would be the right step up.

### 2. LLM Choice — Claude (Anthropic)

Claude was chosen because it: reliably follows structured JSON output instructions without markdown fences when explicitly asked, handles nuanced scope constraints well (refuse legal questions, but answer what a test measures), and maintains conversation state across turns without drifting. The model used is `claude-sonnet-4-20250514` — fast and cost-effective for multi-turn interactions within the 30-second timeout.

### 3. Output Validation — Server-Side URL Grounding

All recommendations from the model are validated against the catalog before being returned. Any URL not in the catalog is silently dropped. This is a hard guarantee: the hard eval requirement ("items from catalog only") is enforced in code, not just in the prompt. This double-layer approach (prompt constraint + code enforcement) makes hallucination-proof responses unconditional.

### 4. Prompt Design — JSON-First with Behavioral Rules

The system prompt encodes the agent's decision logic as numbered rules:
- Rule 2 (clarify first) prevents recommendations on vague turn-1 queries
- Rule 3 (catalog-only) combined with server-side validation ensures grounding
- Rule 7 (end_of_conversation only on user confirmation) prevents premature closing
- Rule 1 (scope) covers refusals for off-topic, legal, and injection attempts

The prompt also includes behavioral notes (OPQ32r as default for professional roles, Graduate Scenarios for graduates, etc.) derived from studying the 10 sample conversations. These notes encode the "expert recommender" persona needed to pass behavior probes.

### 5. Stateless API with Turn Cap

The API is fully stateless — the full conversation history is sent on every POST /chat. Messages beyond 8 are trimmed from the front, honoring the evaluator's turn cap without crashing.

---

## Retrieval Setup

No vector retrieval is used. The catalog summary is ~8KB of text, comfortably within a 200K-token context window. The model has full visibility into all assessments on every turn, which means it can reason about which items to add or remove during refinement without a retrieval step that might miss relevant items.

---

## Evaluation Approach

### Hard Evals (Must Pass)
- Schema compliance validated with Pydantic models — FastAPI enforces field types automatically
- URL grounding: `parse_agent_response()` strips any URL not in `{item["link"] for item in CATALOG}`
- Turn cap: messages trimmed to last 8 before API call

### Behavior Probes (Test Coverage)
The test suite (`tests/test_agent.py`) covers:
- Vague queries → clarifying question, empty recommendations
- Off-topic (legal, general HR, prompt injection) → refusal, empty recommendations
- Specific queries → non-empty recommendations with valid URLs
- Refinement → removing a category updates the shortlist
- `end_of_conversation` is false on early turns
- Max 10 recommendations hard cap

### What Didn't Work / Iterations
- **Initial attempt: full JSON catalog in prompt** — too verbose, the model would sometimes fixate on the first few items. Switching to the condensed one-line-per-item format significantly improved coverage across the catalog.
- **end_of_conversation triggering too early** — adding explicit Rule 7 ("ONLY when user confirms satisfaction") fixed this.
- **Markdown fences in JSON output** — the model occasionally wraps JSON in ```json fences despite instructions. The `parse_agent_response()` function strips these as a safety net.
- **URL hallucination** — even with explicit catalog URLs in the prompt, the model occasionally generated slightly wrong paths. Server-side validation makes this a non-issue for scoring.

---

## Tools Used

- **FastAPI** — lightweight, async, auto-generates OpenAPI docs
- **Anthropic Python SDK (via httpx)** — direct API calls for full control
- **Pydantic** — response schema enforcement
- **Pytest + TestClient** — integration and unit tests
- **python-dotenv** — `.env` file loading for local dev
- **AI-assisted development** — Claude used for code generation, iterating on the system prompt, and drafting the test suite. All design decisions and trade-offs were reviewed and understood manually.
