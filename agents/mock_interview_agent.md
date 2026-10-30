# 🎙️ Mock Interview Agent

## What it does

The Mock Interview Agent **interviews you out loud** as a senior team member at the company you are applying to.
It asks the 7 questions the Summary Agent wrote, listens to your spoken answers, and transcribes them.

## Two voice paths, one prompt

| | **Live (main path)** | **Turn by turn (fallback)** |
|---|---|---|
| How | Gemini Live API: real-time audio both ways over a WebSocket | Question spoken with TTS → you record your answer → it is transcribed |
| Feels like | A real video call — you can interrupt | A walkie-talkie |
| Model | `gemini-2.5-flash-native-audio-preview-12-2025` | `gemini-2.5-flash-preview-tts` + `gemini-2.5-flash` |
| Code | `backend/voice/live_proxy.py` | `backend/voice/turn_based.py` |
| Switch | `VOICE_MODE=live` in `.env` | `VOICE_MODE=turn`, or the toggle on the interview screen |

## How it knows which answer belongs to which question

The agent has two tools (function calling):

- `next_question(number)` — called right **before** asking each question. The server marks the current question,
  and every transcript fragment of your speech after that is saved to it.
- `end_interview()` — called after the goodbye. The browser lets the audio finish, then hands off to review.

```
Browser mic ─PCM 16 kHz─▶ FastAPI WebSocket ─▶ Gemini Live ─▶ voice PCM 24 kHz ─▶ Browser speaker
                                   │
                    next_question(3) → transcript of your speech saved to question 3
```

The API key stays on the server: the browser only talks to FastAPI.

## Long sessions

7 questions take about 15–20 minutes. A Live audio session normally stops at 15 minutes and a connection lasts about
10, so the config turns on **context window compression** and **session resumption**, and the relay reconnects
automatically and continues from the first unanswered question (as does **Resume interview**).

> **Model note:** the Live model is pinned to `gemini-2.5-flash-native-audio-preview-12-2025`, and every session
> starts with a short *spoken* cue ("Hi, I'm ready to start the interview" — `backend/voice/cues/`) instead of a text
> message. On Gemini 2.5 with tools, a session started by text drops with error 1007 a few seconds in.

## How it talks to the other agents

```
Summary Agent ──7 questions──▶ Mock Interview Agent ──question + transcript pairs──▶ Review Agent
```

## System prompt

`backend/agents/mock_interview.py` appends the session details after the system prompt: the company, the role, and the
numbered questions (`interview_brief`).

<!-- PROMPT:START -->
```text
You are the Mock Interview Agent of Mentor AI. You run a realistic spoken mock interview. You play a senior team member at the hiring company: the kind of experienced engineer or manager who sits on the interview panel for this role. The session details below give you the company, the role, and the exact questions.

How to interview:
- This is a voice conversation. Speak naturally and keep each turn short: one question at a time, in 1 to 3 sentences.
- Ask the questions in the given order, in your own natural words, without changing what they test. Do not add new main questions.
- After an answer, you may ask at most one short follow-up if the answer is vague, for example "What was your specific part in that?". Then move on.
- Do not teach, hint, grade, or give feedback during the interview. A neutral "Thanks" or "Got it" is enough. Feedback comes later from the Review Agent.
- If the candidate asks you to repeat or clarify a question, do it briefly. If they say they do not know, reassure them in one sentence and move to the next question.
- Stay in your role as the interviewer. Describe the team and role only with facts from the session details; never invent company facts, people's names, or salary information. If asked something you do not know, say the recruiter will follow up.
- Be professional and friendly: realistic pressure, never hostile.
- Keep the interview moving, about 2 to 3 minutes per question.
```
<!-- PROMPT:END -->
