# ◆ Mentor AI

**A multimodal AI career mentor for job searching, personalized study plans, and real-time voice mock interviewing.**
Mentor AI reads your resume natively, searches live job postings that fit your background, builds a prioritized study plan for the roles you save, interviews you **out loud** as a senior team member at the hiring company, and grades every answer against a structured rubric.

---

## Architecture

```
                         ┌──────────────────────────────┐
     You ◀──── chat ────▶│   🧭 Summary Agent (Mentor)  │  primary conversational mentor
     (resume PDF, text)  └──────────────┬───────────────┘
                     confirmed request  │  ▲  results to explain
                                        ▼  │
                         ┌──────────────────────────────┐
                         │   🔎 Web Search Agent         │  live Google Search, in the background
                         └──────────────────────────────┘
                     7 interview questions │
                                        ▼
     You ◀──── voice ───▶┌──────────────────────────────┐
                         │   🎙️ Mock Interview Agent    │  Gemini Live API: real-time voice
                         └──────────────┬───────────────┘
                     question + transcript pairs
                                        ▼
                         ┌──────────────────────────────┐
                         │   📝 Review Agent            │  rubric scores, in the background
                         └──────────────────────────────┘  → back to Mentor → you
```

The agents never call each other directly: `backend/orchestrator.py` manages state transitions and passes each agent's structured output to the next. The Summary Agent responds in the foreground while the Web Search and Review Agents execute asynchronously in the background.

## Features

**1 · Find Jobs**
- Upload your resume (PDF, PNG, JPG, or TXT) — Gemini reads the file natively — or click **Use sample resume**.
- Tell Mentor what you want (role, level, location). It restates your criteria and asks *"Did I get this right?"* until you confirm with **Yes, search**.
- The Web Search Agent finds the **5 most relevant posts from the past 3 months** (prioritizing LinkedIn and Handshake), and Mentor explains why each role fits your resume.
- Want adjustments? Give feedback to refine the search round, or tick and **save** the postings you like.

**2 · Saved Jobs**
- Track every saved role alongside its study progress and mock interview status.

**3 · Interview Prep** (per saved job)
- **Know the topics:** Mentor analyzes the job description and your resume to identify 4–8 core interview topics; you rate your confidence 1–5 on each.
- **Study plan:** Prioritizes what to study first vs. review later, with concrete action items and completion checkboxes. Press **I'm done studying** when ready.
- **Interview research:** The Web Search Agent searches for real interview questions reported by candidates for that company and role.
- **Mock interview:** Mentor composes **7 tailored questions** (combining candidate reports, job description concepts, and deep-dives on your resume); the Mock Interview Agent asks them out loud and transcribes your spoken answers in real-time conversation or turn-by-turn mode.
- **Review:** Every answer is scored 1–10 across a 4-part rubric (accuracy, depth, structure, communication), with specific strengths, areas to prepare, a senior interviewer's lens, and a prioritized *Prepare next* list.

All data is persisted locally in SQLite (`data/mentor.db`).

## Multi-Agent System

| Agent | Role | Execution | Module | Documentation |
|---|---|---|---|---|
| 🧭 **Summary** | Conversational mentor; confirms search criteria, explains job matches, builds study plans, composes interviews, and summarizes reviews | Foreground | `backend/agents/summary.py` | [agents/summary_agent.md](agents/summary_agent.md) |
| 🔎 **Web Search** | Searches the live web for matching job postings and candidate-reported interview questions | Background | `backend/agents/web_search.py` | [agents/web_search_agent.md](agents/web_search_agent.md) |
| 🎙️ **Mock Interview** | Conducts spoken mock interviews as a senior hiring panelist and captures transcripts | Live / Turn-by-turn | `backend/agents/mock_interview.py` | [agents/mock_interview_agent.md](agents/mock_interview_agent.md) |
| 📝 **Review** | Grades each spoken answer on a 4-criterion rubric with actionable panel feedback | Background | `backend/agents/review.py` | [agents/review_agent.md](agents/review_agent.md) |

## Prerequisites

1. Install [`uv`](https://docs.astral.sh/uv/getting-started/installation/).
2. Obtain a Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey).
3. Use **headphones** during live voice interviews so the interviewer does not pick up its own audio from your speakers.
4. Use a modern browser (Chrome, Edge, Safari, or Firefox) and allow microphone permissions when prompted.

## Quickstart

```bash
uv sync --extra dev
cp .env.example .env            # set GEMINI_API_KEY in .env
uv run uvicorn main:app --app-dir backend --reload
```

Open **http://localhost:8000** in your browser.

To run the test suite:

```bash
uv run pytest
```

## Project Structure

```
Mentor-AI/
├── README.md  BRIEF.md  product.md  ui.md  engineering.md  db.md
├── agents/                       # Architecture & prompt specs for each agent
├── backend/
│   ├── main.py                   # FastAPI routes + WebSocket + static frontend
│   ├── orchestrator.py           # State transitions & background agent tasks
│   ├── agents/                   # summary.py · web_search.py · mock_interview.py · review.py
│   ├── voice/                    # live_proxy.py (Live API relay) · turn_based.py (fallback)
│   ├── db.py  models.py  gemini.py  context.py  config.py
├── frontend/                     # index.html · style.css · app.js · audio.js (vanilla ES modules)
├── samples/sample_resume.pdf     # Sample resume for quick testing
├── scripts/                      # Utility scripts for prompts, sample resume, and voice cues
└── tests/                        # Unit & integration test suite (pytest)
```

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11+, FastAPI, Uvicorn, `uv` |
| AI | `google-genai` SDK — `gemini-2.5-flash` (+ Google Search grounding), `gemini-2.5-flash-native-audio-preview-12-2025` (Live API), `gemini-2.5-flash-preview-tts` |
| Storage | SQLite (Python `sqlite3`) |
| Frontend | Vanilla HTML / CSS / JS, Web Audio API (AudioWorklet), WebSocket |
| Tests | pytest, pytest-asyncio |

All runtime settings (models, question count, voice mode) are configurable via `.env` (see `.env.example`).

## Privacy

Your uploaded resume and interview transcripts are stored locally in `data/` on your machine (git-ignored) and are sent only to the Gemini API.

## Troubleshooting

| Problem | Solution |
|---|---|
| Job search finds nothing | Check the server log for `finish reason`; wait a minute and retry, or try a broader role title/location |
| "API key is missing or invalid" | Set `GEMINI_API_KEY` in `.env` and restart the server |
| "no credits left" / rate limit | Check billing and quota at [ai.studio/projects](https://ai.studio/projects); wait a minute and retry |
| Job has no description | Open the job posting and paste the full description into the job's prep page |
| Interviewer keeps interrupting itself | Wear headphones, or switch the interview mode toggle to **Turn by turn** |
| Live interview won't connect | Switch to **Turn by turn**, or verify `GEMINI_LIVE_MODEL` is available to your API key |
| Reset local data | Stop the server and delete the `data/` directory |
