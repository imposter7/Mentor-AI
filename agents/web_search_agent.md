# 🔎 Web Search Agent

## What it does

The Web Search Agent searches the **live web** with Gemini's built-in Google Search grounding (plus URL context to
read pages). It never talks to you — it hands its findings to the Summary Agent.

| Task function | When | What it returns |
|---|---|---|
| `find_jobs` | You press **Yes, search** (or send feedback on results) | The 5 best-fitting posts from the past 3 months: title, company, location, level, link, source, posted date, a summary of the responsibilities and requirements, and a 0–100 fit score |
| `find_interview_reports` | You press **I'm done studying** on a saved job | Interview questions candidates reported for that company and role, with source links |

## When it runs

**In the background.** Searching takes about 30–90 seconds. The app shows *"🔎 Web Search Agent is searching…"* and polls
for the result.

## How it works: search wide, then rank

```
1. Plan      code turns your request into 3–4 search angles      "Entry-level Backend Engineer jobs in New York City", …
2. Search    one Google Search grounded call per angle, in parallel        → findings + real source links
3. Extract   one JSON call turns all findings into candidate posts         → FoundJobs
4. Clean     code: real links only, last 90 days, no duplicates or posts you've already seen
5. Rank      one JSON call scores every candidate 0–100 for fit           → JobRanking
             code adds small nudges (+5 LinkedIn/Handshake, −10 unknown date) and keeps the top 5
6. Broaden   fewer than 3 left? code searches once more with a wider level and location
```

**Why it is built this way**
- **Gemini will not copy web pages word for word.** A request to "copy the job description" comes back empty with
  finish reason `RECITATION`. Every search asks for descriptions *in the model's own words*, and an empty or blocked
  answer is retried once with a stronger reminder.
- **Several simple searches beat one clever one.** Site and date operators (`site:`, `after:`) starve the results,
  so the angles are plain phrases and code checks dates afterwards.
- **"Most relevant" needs a pool to choose from.** The agent collects up to ~20 candidates and ranks them instead of
  returning the first five it finds.
- **No URL context tool.** Reading full pages cost ~120k tokens and ~2 minutes per search; summaries from search are
  enough to rank, and you can paste the full post into a saved job.
- Gemini 2.5 cannot use Google Search and a forced JSON schema in the same call, so searching and extracting are
  separate calls.

## How it talks to the other agents

```
Summary Agent ──confirmed request + your feedback──▶ Web Search Agent ──5 posts──▶ Summary Agent (explains them)
Saved job ──────────────────────────────────────────▶ Web Search Agent ──reported questions──▶ Summary Agent (writes the interview)
```

## Inputs and outputs

| | |
|---|---|
| **Input** | The confirmed request (`JobPreferences`), feedback on earlier rounds, posts already shown — or one saved job |
| **Output** | `FoundJobs`, `JobRanking`, `InterviewResearch` in `backend/models.py` |
| **Model** | `gemini-2.5-flash` + the `GoogleSearch` tool |

## Honest limits (live search only)

- **LinkedIn** postings are preferred when Google Search shows them, but many results come from company career pages
  and job boards (Built In, Greenhouse, Lever…). Those are labelled `other`.
- **Handshake** is behind a login, so Google Search almost never sees it. Found a Handshake post you like? Save any
  similar result and paste the Handshake description into it — or add your own post's description.
- **Posted dates** are not always shown. Unknown dates appear as "date unknown" and rank a little lower.
- **Descriptions are summaries.** Interview prep works from them; pasting the full post on the job card makes the
  topics and questions sharper.

## System prompt

Configured in `backend/agents/web_search.py`:

<!-- PROMPT:START -->
```text
You are the Web Search Agent of Mentor AI. You work in the background for the Summary Agent, which talks to the user. You search the live web with Google Search and report only what you actually find.

Finding job posts:
- Search for open postings that match the user's role, level, and location. LinkedIn postings are preferred when they appear. Handshake postings are usually behind a login, so include them only when a public page shows up. Company career pages and public job boards are fine; report which site each post is on.
- Only use pages that are publicly visible. Never try to log in or get around a login wall.
- Prefer posts published inside the date window you are given. If a post does not show its date, report the date as unknown; never guess a date.
- Skip posts that are closed, duplicated, or already shown to the user.
- Describe every post in your own words: summarize its responsibilities and requirements. Never copy text from a web page word for word; verbatim copies are blocked and the whole answer is lost.

Ranking job posts:
- Judge fit honestly: role and level, location and work mode, the user's must-haves and deal-breakers, their feedback, and overlap with the strengths in their resume. A well-known company is not a better fit by itself.

Researching interview questions:
- Look for interview questions that candidates reported for the specific company and role: public interview reviews, forums, and blog posts.
- Only report questions that a source actually says were asked. If you find nothing reliable, say so; never make questions up.

Always:
- Every URL you give must come from your search results. Never build or guess a URL.
- Follow the TASK instructions in each request exactly.
```
<!-- PROMPT:END -->
