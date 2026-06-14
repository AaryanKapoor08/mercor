# AI Interviewer

A voice based technical interviewer. A candidate enters their GitHub profile, the system reads their public repositories to understand their background, and then runs a spoken interview in the browser. When the interview ends, the conversation is scored and the candidate receives written feedback.

## How it works

1. The candidate submits their GitHub URL on the start screen.
2. The backend reads the candidate's public repositories and stores a summary of them as interview context.
3. The browser opens a live voice connection. The interviewer speaks through OpenAI's realtime voice model, and the questions are shaped by the candidate's GitHub activity.
4. The candidate's speech is transcribed in real time and saved alongside the interviewer's turns, building a full transcript.
5. When the candidate ends the session, the transcript is sent to a language model that returns a score out of ten and feedback.

## Architecture

The project is a monorepo with two applications and a set of shared packages.

```
mercor/
  apps/
    backend/      Express API, realtime session handling, scoring
    frontend/     React single page app served by Bun
  packages/
    ui/                 Shared React components
    eslint-config/      Shared lint rules
    typescript-config/  Shared TypeScript settings
```

### Request flow

```
Browser (React)
   |
   |  1. POST /api/v1/pre-interview  (GitHub URL)
   v
Backend (Express)
   |  reads public repos through a proxy, stores interview context in Postgres
   |
   |  2. POST /api/v1/session/:id  (WebRTC offer)
   v
OpenAI Realtime API (voice interviewer)
   |
   |  audio flows directly to the browser over WebRTC
   |  a parallel WebSocket on the backend records the interviewer's transcript
   |
Browser
   |  microphone audio is transcribed live and the candidate's
   |  answers are posted back and stored
   |
   |  3. GET /api/v1/result/:id  (after the call ends)
   v
Backend scores the full transcript with a language model and returns
the score, feedback, and transcript.
```

### Backend

The backend is an Express service that owns four responsibilities.

| Concern | Where | Notes |
| --- | --- | --- |
| GitHub context | `scrapers/github.ts` | Reads public repositories through an outbound proxy |
| Realtime session | `index.ts`, `sideband.ts` | Negotiates the WebRTC connection with OpenAI and records the interviewer side over a WebSocket |
| Persistence | `db.ts`, `prisma/` | Prisma client over PostgreSQL |
| Scoring | `result.ts` | Sends the transcript to a language model and parses a structured result |

The data model has two tables. An `Interview` holds the GitHub context, status, score, and feedback. A `Message` holds a single turn of the conversation and references its interview.

### Frontend

The frontend is a React single page application served directly by Bun. It has three screens.

| Route | Screen | Purpose |
| --- | --- | --- |
| `/` | Start | Collect the GitHub URL and create the interview |
| `/interview/:id` | Interview | Run the live voice call and show speaking activity |
| `/result/:id` | Result | Show the score, feedback, and transcript |

During the interview the browser opens a WebRTC connection for voice, captures the microphone, and streams it to a speech to text service so the candidate's answers are transcribed and saved as they speak. Two voice meters show who is speaking.

## Tech stack

| Area | Technology |
| --- | --- |
| Language | TypeScript |
| Runtime and package manager | Bun |
| Monorepo tooling | Turborepo |
| Frontend framework | React 19 with React Router |
| Styling | Tailwind CSS with Radix UI primitives |
| Backend framework | Express 5 |
| Database | PostgreSQL with Prisma |
| Voice interview | OpenAI Realtime API over WebRTC |
| Speech to text | Deepgram |
| Scoring | Google Gemini |
| Validation | Zod |

## Getting started

### Prerequisites

You will need Bun installed, a PostgreSQL database, and API access for OpenAI, Deepgram, and Google Gemini.

### Install

```bash
bun install
```

### Environment

The backend reads the following values from the environment:

| Variable | Used for |
| --- | --- |
| `DATABASE_URL` | PostgreSQL connection string |
| `OPENAI_KEY` | OpenAI realtime voice |
| `GEMINI_API_KEY` | Transcript scoring |
| `PROXY_URL` | Outbound proxy for reading GitHub |

The frontend reads the backend address from its own configuration in `apps/frontend/src/lib/config.ts`.

### Database

```bash
cd apps/backend
bunx prisma migrate deploy
bunx prisma generate
```

### Run

From the repository root:

```bash
bun run dev
```

This starts both applications through Turborepo. The backend listens on port 3001 and the frontend is served by Bun.

## Scripts

| Command | Action |
| --- | --- |
| `bun run dev` | Start every app in development |
| `bun run build` | Build every app |
| `bun run lint` | Lint the workspace |
| `bun run check-types` | Type check the workspace |
| `bun run format` | Format the source files |

## Status

This is a working build that follows the flow above. A few rough edges remain, such as GitHub URL validation and the frontend speech to text key handling, and they are marked in the source.
