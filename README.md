# TalentPulse AI

 live application url : [talentpulseAI - Automating Hiring](https://talentpulse.slicearrow.com/admin)
TalentPulse AI is a recruitment application for collecting candidate applications, scheduling AI-led voice interviews, and reviewing interview results. Candidates browse roles and upload a PDF resume. Recruiters use a dashboard to prepare questions, schedule an interview, review the transcript, and generate an evaluation.

## What it does

- Shows job openings and an application form at `/apply`.
- Stores candidate details, job descriptions, PDF resumes, bookings, question banks, interview events, transcripts, and evaluations.
- Generates a default question bank for each a[text](https://talentpulse.slicearrow.com/admin)pplication; recruiters can request tailored questions through an n8n workflow.
- Schedules a LiveKit interview and creates a candidate join link at `/interview?token=...`.
- Runs a voice interviewer using LiveKit Agents and Sarvam AI for speech recognition, conversation, and speech synthesis.
- Lets recruiters manage candidates, bookings, interview data, and evaluations at `/admin`.
- Sends interview completion events to n8n through an optional retrying outbox worker.

## Architecture

```text
React + Vite frontend
  ├─ Candidate application and LiveKit interview room
  └─ Recruiter dashboard
           │ HTTP
           ▼
FastAPI backend ────────  PostgreSQL
  ├─ Applications, bookings, room tokens, transcripts, evaluations
  ├─  n8n webhooks: questions, scheduling, email, evaluation
  └─  outbox worker: interview completion handoffs
           │
           ▼
LiveKit Cloud agent (separate process) ── Sarvam AI
```

The API and LiveKit agent run together for local development. In production, the API runs as a Render web service and the agent runs separately on LiveKit Cloud. Both must use the same PostgreSQL database.

| Area | Technology |
| --- | --- |
| Frontend | React 18, Vite 5, LiveKit React components |
| API | Python 3.12, FastAPI, Uvicorn |
| Storage | SQLite locally; PostgreSQL for separate services |
| Voice interview | LiveKit Agents, WebRTC, Sarvam AI |
| Workflow integration | n8n webhooks |
| Deployment files | `render.yaml`, `backend/Dockerfile`, `frontend/netlify.toml` |

The automation starts when a candidate applies through the React and Vite frontend. FastAPI validates the application, stores it in  PostgreSQL, and creates an initial question bank. Recruiters can use n8n to generate questions tailored to the resume and job description, schedule the interview, and email the join link. When the candidate joins, LiveKit connects them to a voice agent that uses Sarvam AI to turn speech into text, guide the conversation, and speak its responses. The backend saves the transcript and interview events; an n8n evaluation workflow then produces a scorecard and ranking for the recruiter dashboard. An optional outbox worker retries completion notifications if a webhook is temporarily unavailable.

## Sarvam AI models and AI techniques

The LiveKit interview worker configures these Sarvam models for Indian English (`en-IN`):

| Model | Role in the interview |
| --- | --- |
| `saaras:v4` | Speech-to-text (STT): transcribes the candidate's spoken answers. |
| `sarvam-105b-conversations` | Large language model (LLM): conducts the conversation, asks planned questions, and responds to the candidate. It is configured with temperature `0.2`. |
| `bulbul:v3` | Text-to-speech (TTS): speaks the interviewer's replies using the `shubh` voice. |

The application uses a **speech-to-text → LLM → text-to-speech voice pipeline** to conduct a real-time interview. It uses **contextual prompting** to give the LLM the candidate's resume summary, skills, and assigned question bank. A **rule-based conversation controller** keeps questions in order, allows at most one follow-up per question, and tracks progress across reconnects. Recruiters can request **resume- and job-description-based question generation** through an n8n workflow; the backend removes duplicate questions before saving the bank. After an interview, an n8n **transcript evaluation** workflow returns per-question feedback and answer statuses. The backend then normalizes those results, calculates category and overall scores, and ranks candidates. 

## Run locally

You need Python 3.12, Node.js 18+ with npm, and LiveKit and Sarvam credentials to conduct a real voice interview. The application and dashboard can be explored without starting an interview. The backend creates its database schema on startup; without `DATABASE_URL`, it uses `backend/data/interviewer.db`.

1. Create `backend/.env` with local settings. Replace the placeholders with your own credentials:

   ```dotenv
   DATABASE_URL=sqlite:///./data/interviewer.db
   BACKEND_PUBLIC_URL=http://localhost:3000
   FRONTEND_PUBLIC_URL=http://localhost:5180
   CORS_ALLOWED_ORIGINS=http://localhost:5180
   INTERNAL_SERVICE_SECRET=replace-with-a-random-secret
   TOKEN_SECRET=replace-with-another-random-secret
   LIVEKIT_URL=wss://your-project.livekit.cloud
   LIVEKIT_API_KEY=your-livekit-key
   LIVEKIT_API_SECRET=your-livekit-secret
   SARVAM_API_KEY=your-sarvam-key
   N8N_EMAIL_URL=
   N8N_SUMMARY_URL=
   N8N_GENERATE_QUESTIONS_URL=
   ```

   The last three entries disable the configured default webhook URLs during local development. Add your n8n URLs when those workflows are available. Keep real secrets out of Git.

2. Install and start the backend from a terminal:

   ```bash
   cd backend
   python -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   python run_local.py
   ```

   `run_local.py` starts both the FastAPI server on port 3000 and the LiveKit worker. To run only the API, use `uvicorn app.main:app --host 0.0.0.0 --port 3000 --reload` from `backend/` instead.

3. Install and start the frontend in another terminal:

   ```bash
   cd frontend
   npm install
   npm run dev
   ```

   The Vite server uses port 5180. Open `http://localhost:5180/apply` for the candidate flow or `http://localhost:5180/admin` for the dashboard. API health is at `http://localhost:3000/health`, and interactive API documentation is at `http://localhost:3000/docs`.

### Frontend configuration

Create `frontend/.env.local` if the API is not at the default `http://127.0.0.1:3000`:

```dotenv
VITE_BACKEND_URL=http://localhost:3000
```

`VITE_APPLICATION_URL` can override the application submission endpoint. `VITE_JOB_DESCRIPTION` supplies a default description for a custom role link. Vite embeds these values in the browser bundle, so do not put secrets in `VITE_` variables.

## Typical workflow

1. A candidate selects a role at `/apply`, enters contact and profile details, chooses an interview difficulty, and uploads a PDF resume smaller than 5 MB.
2. The API stores the application and creates a starter question bank. The role list is currently defined in `frontend/src/main.jsx`.
3. A recruiter opens `/admin`, reviews the candidate, and optionally generates a tailored question bank through n8n.
4. The recruiter schedules an available slot. The API freezes the current question bank onto the booking and creates a LiveKit join link. An optional n8n workflow can create the room and send the invitation email.
5. The candidate opens the join link. LiveKit starts the named `interview-bot` agent when the candidate connects; the agent conducts the interview and saves its transcript and events.
6. After a transcript is available, the recruiter requests an evaluation. The API sends the interview data to n8n, stores the returned scorecard, and calculates candidate rankings.

Question generation and evaluation require working n8n webhook endpoints. Scheduling can use the API's built-in LiveKit flow when `N8N_SCHEDULE_URL` is unset.

## Configuration reference

Backend variables are read from `backend/.env` locally and from the service environment in deployment.

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL` | `sqlite:///...` for local use or `postgresql://...` for shared storage. |
| `INTERNAL_SERVICE_SECRET` | Protects `/internal/*` routes and the full application reset endpoint. |
| `TOKEN_SECRET` | Signs application booking and join tokens. |
| `BACKEND_PUBLIC_URL` | Public API URL used by the separate LiveKit worker. |
| `FRONTEND_PUBLIC_URL` | Frontend origin used to create candidate interview links. |
| `CORS_ALLOWED_ORIGINS` | Comma-separated allowed frontend origins; defaults to `*`. |
| `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` | LiveKit connection and token credentials. |
| `SARVAM_API_KEY` | Sarvam AI credentials for the voice worker. |
| `N8N_SCHEDULE_URL` | Optional room scheduling webhook. |
| `N8N_EMAIL_URL` | Optional invitation email webhook; set to an empty value to disable its configured default. |
| `N8N_GENERATE_QUESTIONS_URL` | Question generation webhook. |
| `N8N_SUMMARY_URL` | Interview evaluation webhook. |
| `N8N_INTERVIEW_COMPLETE_URL` | Optional completion webhook used by the outbox worker. |
| `N8N_WEBHOOK_SECRET` | Optional secret sent to n8n as `X-Webhook-Secret`. |

## Main API routes

| Route | Use |
| --- | --- |
| `GET /health` | Health check. |
| `POST /applications` | Submit candidate details and a PDF resume as multipart form data. |
| `GET /applications`, `GET /applications/{candidate_id}` | List and inspect applications. |
| `POST /applications/{candidate_id}/generate-questions` | Generate a candidate question bank through n8n. |
| `POST /applications/{candidate_id}/schedule` | Reserve a slot and create interview links. |
| `POST /applications/{candidate_id}/send-email` | Send or resend an invitation through n8n. |
| `GET /applications/{candidate_id}/interview` | Read interview session and transcript data. |
| `POST /applications/{candidate_id}/interview-summary` | Generate an evaluation through n8n. |
| `GET /applications/{candidate_id}/interview-summary` | Read the stored evaluation and ranking. |
| `GET /slots` | List available interview slots. |
| `GET /interview/join-info?token=...` | Validate a candidate join token and return room details. |

The API also has delete and internal service routes; see `/docs` and [backend/README.md](backend/README.md) for details. The current dashboard and application endpoints do not implement recruiter login or role-based access control. Add authentication and access controls before exposing candidate records or administrative operations to an untrusted audience.

## Tests and builds

```bash
cd backend
python -m pytest
```

```bash
cd frontend
npm test
npm run build
```

## Deployment

- `render.yaml` defines the API service. It runs `uvicorn app.main:app` and checks `/health`.
- Deploy `backend/` as a LiveKit Cloud agent using the included `Dockerfile`. Keep its agent name `interview-bot`, which is referenced by candidate tokens. Configure `DATABASE_URL`, `SARVAM_API_KEY`, `BACKEND_PUBLIC_URL`, and the same `INTERNAL_SERVICE_SECRET` used by the API.
- Build `frontend/` with `npm run build` and publish `frontend/dist/`. Set `VITE_BACKEND_URL` to the deployed API URL and `FRONTEND_PUBLIC_URL` on the API to the frontend origin.
- Use PostgreSQL for the API and worker so both services can read the same interview data. Configure any n8n webhook URLs on the API and, if using completion handoffs, run `python -m app.outbox_worker` as a separate process from `backend/`.

More deployment and worker details are in [backend/README.md](backend/README.md).
