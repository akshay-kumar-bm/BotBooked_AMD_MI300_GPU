# BotBooked.ai - Agentic Meeting Scheduling Assistant

An agentic AI assistant that reads a meeting request (JSON with email text and attendees), checks participants' Google Calendars, and finds or reschedules slots, using an LLM served by vLLM on an AMD Instinct MI300 GPU. Built as a hackathon submission (AMD).

## What it does
`app.py` implements a LangGraph workflow: an LLM extracts meeting details from natural-language email content, Google Calendar availability is fetched per attendee, and a scheduling algorithm picks optimal times based on per-participant preferences (working hours, max meetings/day, back-to-back avoidance, buffer minutes) and meeting priority, with the ability to move lower-priority events.

## Key features
- LangGraph state graph orchestration (`StateGraph`), JSON-parsed LLM output via LangChain.
- Google Calendar integration (reads per-user OAuth token files from a `Keys/` directory).
- Preference- and priority-aware scheduling.
- Flask `POST /receive` endpoint (`run.py`) that takes the request JSON and returns the final output.
- Streamlit front end (`streamlit_app.py`, ~365 lines) with dark theme to submit requests and view responses.
- `Submission.ipynb` runs the Flask API in a background thread and tests it with curl and sample requests.

## Tech stack
LangGraph, LangChain (`langchain-openai`, `langchain-core`), Qwen3-4B served via vLLM (OpenAI-compatible API) on AMD MI300, Google Calendar API, Flask, Streamlit.

## Project structure
```
app.py              # LangGraph scheduling logic
run.py              # Flask API (/receive, port 5000)
streamlit_app.py    # Streamlit UI
Submission.ipynb    # submission notebook / tests
assets/             # logo, presentation video, images
requirements.txt    # langgraph, langchain-openai, langchain-core
```

## Setup
```bash
pip install -r requirements.txt   # note: Google API and Flask/Streamlit packages are also imported but not listed
python run.py                      # Flask API on :5000
streamlit run streamlit_app.py
```
Requires a running OpenAI-compatible vLLM endpoint and Google OAuth token files in `Keys/` (not included). The model client in `app.py` uses a placeholder API key.

## Limitations
- `run.py` imports `from app1 import graph` but the module is `app.py` (the notebook uses `from app import builder`); fix needed.
- requirements.txt is incomplete.
- Compiled `__pycache__` and a notebook checkpoint are committed; the README mentions a LICENSE file that is absent.
