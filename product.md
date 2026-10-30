# Mentor AI — Product Design Doc

**Status:** v1.0
**One-liner:** A multimodal AI agent that reads your resume, finds live job posts that fit you, builds a study plan, and interviews you out loud as a senior employee of the company — then grades every answer.

---

## 1. The user & the moment

- **Who:** A student or early-career job seeker (searching on LinkedIn, Handshake, and career pages) with a resume PDF and too many tabs open.
- **When:** Preparing for upcoming applications and interviews. They know roughly what they want, but searching, deciding what to study, and practicing interviews are three separate chores.
- **Why now:** Gemini can read a PDF natively, search the live web with grounding, and hold a real-time voice conversation. One coordinated multi-agent system covers the full path from *"what should I apply to?"* to *"how did my interview go?"*.

## 2. The contract (I/O)

- **Input:** Resume (PDF / PNG / JPG / TXT), a typed job request, a 1–5 self-rating per topic, and spoken answers.
- **Output:** 5 job posts per round with links and reasons, a prioritized study plan, a 7-question voice interview, and a per-question review (rubric scores, strengths, what to prepare, what a senior interviewer listens for).
- **The loops:**
  1. Request → Mentor restates → you refine → **Yes, search** → 5 posts → feedback → 5 new posts → **Save**.
  2. Saved job → topics → self-ratings → study plan → **I'm done studying** → research + interview composed → voice interview → review.

## 3. The core experience

> "It found a real posting I'd missed, then a 'senior engineer' from that company interviewed me out loud with a question people actually got asked there — and told me exactly what I fumbled."

## 4. Scope: what we ARE building (v1)

- Resume upload (or a one-click sample resume) read natively by Gemini.
- Job search chat with a confirmation loop and a feedback loop; save the posts you choose.
- Saved Jobs tab with study and interview status per job.
- Topic list → self-ratings → study plan with study-first / review-later and per-topic done checkboxes.
- Interview research (reported questions with sources) → 7-question interview (configurable 5–10).
- Voice mock interview: Gemini Live API, with a turn-by-turn fallback.
- Review with a 4-criterion rubric and an overall 1–10 score per question, plus "prepare next".
- Everything stored locally in SQLite.

## 5. Scope: what we are NOT building

- **No accounts or auth** — single local user, zero onboarding.
- **No cloud database** — resumes are personal data; they stay in `data/` on the user's machine.
- **No scraping behind logins** — public pages via Google Search grounding only.
- **No static job dataset** — job data is live search only; the fallback for missing descriptions is pasting the description.
- **No comparison across interview attempts** — each review stands alone.
- **No video / body-language analysis, no resume rewriting, no cover letters, no auto-apply.**

## 6. The signature detail

**The agent rail.** Four chips (🧭 Summary, 🔎 Web Search, 🎙️ Mock Interview, 📝 Review) show each agent's live state (*ready → working*), making the multi-agent orchestration transparent to the user.

## 7. Success metrics

- **Primary:** Complete a full end-to-end voice interview and receive a structured multi-criterion rubric review.
- **Secondary:** ≥80% of job search rounds return at least 3 relevant postings with verified links.

## 8. Risks & mitigations

- **Live search variance** — posts close, LinkedIn/Handshake limit public pages, dates are missing. Mitigation: show "date unknown", keep only links from real search results, paste-the-description fallback, round-by-round feedback.
- **API quota limits** — clear error handling for rate limits and billing/quota exhaustion.
- **Room noise and echo** — headphones recommendation in the UI plus a turn-by-turn voice fallback.

## 9. Architecture alignment

- **UX:** Three tabs, one chat, and a prep page that reveals steps top-to-bottom (`ui.md`).
- **Engineering:** Code owns status transitions; agents are stateless functions; background agents never block an HTTP request (`engineering.md`, `db.md`).
