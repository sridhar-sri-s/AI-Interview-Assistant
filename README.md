# AI Interview Assistant

A voice-based mock interviewer. Pick a topic, and an AI interviewer named **Natalie** asks you spoken questions that adapt to what you actually said. Answer out loud, and at the end get a score out of 5 with strengths and areas to improve.

## Features

- **Six topics:** Self Introduction, Generative AI, Python, English, HTML, CSS
- **Voice in, voice out:** answers are recorded in the browser; questions are spoken back in a natural voice
- **Adaptive follow-ups:** each question builds on your previous answer, and gets simpler if you're stuck
- **Streaming audio:** Natalie starts speaking before the full audio has been generated
- **Feedback report:** a score out of 5, specific strengths, and areas of improvement drawn from your real answers

## How it works

```
Browser (frontend/)                Flask server (backend/app.py)          Cloud services
───────────────────                ─────────────────────────────          ──────────────
Pick topic ──POST /start-interview──► reset memory, ask question 1 ──────► OpenRouter (Gemini 2.5 Flash)
                                                                  ──────► Murf AI (text → speech)
Play voice ◄──── base64 MP3 stream ──┘

Record answer ──POST /submit-answer─► transcribe the answer ────────────► AssemblyAI (speech → text)
                                     ask the next question ─────────────► OpenRouter
Play voice ◄──── base64 MP3 stream ──┘                         ─────────► Murf AI

End interview ──POST /get-feedback──► score + feedback as JSON ─────────► OpenRouter
```

- **Memory:** a LangChain agent with LangGraph's `InMemorySaver` keeps the full conversation, so follow-ups and feedback can reference earlier answers.
- **Streaming:** Murf's MP3 chunks are sent to the browser as base64 text lines and played through the `MediaSource` API as they arrive.

## Tech stack

| Layer | Technology |
| --- | --- |
| Frontend | HTML, JavaScript, Tailwind CSS (CDN), Font Awesome |
| Backend | Python, Flask, Flask-CORS |
| LLM | Gemini 2.5 Flash via [OpenRouter](https://openrouter.ai), LangChain + LangGraph |
| Speech-to-text | [AssemblyAI](https://www.assemblyai.com) |
| Text-to-speech | [Murf AI](https://murf.ai) |

## Project structure

```
AI-Interview-Assistant/
├── backend/
│   ├── app.py        # Flask API: interview flow, LLM, speech services
│   └── .env          # your API keys (not committed)
└── frontend/
    ├── index.html    # UI layout
    └── index.js      # recording, audio playback, API calls
```

## Getting started

### Prerequisites

- Python 3.10+
- A modern browser with microphone access (Chrome recommended)
- API keys for [OpenRouter](https://openrouter.ai/keys), [AssemblyAI](https://www.assemblyai.com/dashboard), and [Murf AI](https://murf.ai/api)

### 1. Set up the backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install flask flask-cors python-dotenv langchain langchain-openai langgraph assemblyai requests
```

Create `backend/.env`:

```env
OPENROUTER_API_KEY=your_openrouter_key
ASSEMBLYAI_API_KEY=your_assemblyai_key
MURF_API_KEY=your_murf_key
```

Start the server:

```bash
python app.py
```

It runs at `http://127.0.0.1:5000`. Visiting that URL should return `{"status": "Backend is running"}`.

### 2. Run the frontend

In a second terminal:

```bash
cd frontend
python -m http.server 8000
```

Open [http://localhost:8000](http://localhost:8000) and allow microphone access when prompted.

## Usage

1. Choose a topic from the sidebar.
2. Click **Start Interview** and listen to Natalie's first question.
3. Click the **mic** to record your answer, click again to stop, then **Submit Answer**.
4. Repeat for each question.
5. Click **End Interview** to get your score and feedback.

## API

| Method | Endpoint | Body | Returns |
| --- | --- | --- | --- |
| `GET` | `/` | — | Health check |
| `POST` | `/start-interview` | `{"subject": "Python"}` | Streamed base64 MP3 of the first question |
| `POST` | `/submit-answer` | `multipart/form-data` with an `audio` file (webm) | Streamed base64 MP3 of the next question; `X-Question-Number` header |
| `POST` | `/get-feedback` | `{}` | `{"success": true, "feedback": {"subject", "candidate_score", "feedback", "areas_of_improvement"}}` |

## Known limitations

- **One interview at a time:** conversation state is global, so simultaneous users share a session.
- **Memory isn't saved:** interviews are lost when the server restarts.
- **Manual ending:** the interview doesn't stop on its own after 5 questions; use **End Interview**.
- **Question counter:** custom response headers aren't exposed through CORS yet, so the counter may stay at 1.

## Roadmap

- Per-user sessions keyed by a unique `thread_id`
- One LLM call per answer instead of two
- Automatic end after the final question
- Schema-validated feedback JSON
- Saved interview history and progress tracking
- `requirements.txt` and tests
