# AI Marker for Primary Schools

An end-to-end assignment platform that lets primary school teachers create rubric-based assignments, distribute them to students, and **auto-mark up to 30 student submissions per batch using a two-stage LLM pipeline** that scores against the teacher's rubric and generates personalised, NSW-curriculum-aligned written feedback.

Built as a capstone project for the UTS Software Development Studio (SDS) program.

> *Source code and application screenshots are not shared publicly out of respect for the client engagement. This README documents the architecture, design decisions, and my contributions; happy to walk through the implementation.*

---

## Overview

Marking is one of the most time-consuming tasks in primary teaching — especially for short-answer and extended-writing questions where rubric-based judgement is required. This platform replaces the manual marking pass with an AI marker that:

- Reads each student's answers question-by-question
- Scores each answer against the teacher's own rubric criteria and mark allocations
- Generates a short, warm, teacher-style comment per question
- Produces a holistic feedback paragraph for the whole assignment
- Writes results back to the database for the student and teacher to review and edit

Teachers remain in control — every AI-generated grade and comment is editable in the UI before being released to students.

## Key Features

- **Role-based access** — Firebase Auth with separate Teacher and Student dashboards routed by Firestore role lookup on login
- **Assignment builder** — teachers create multi-question assignments and assign them to a class
- **Rubric builder** — per-question rubric with named criteria, descriptions, and mark allocations; total marks calculated automatically
- **Student submission flow** — students log in, view active assignments, and submit answers inline
- **Batch AI marking** — single click marks up to 30 submissions in one API call, using rubric criteria as the scoring rubric for the LLM
- **Grade review and edit** — teachers can adjust scores, criteria breakdowns, and feedback before publishing
- **Submission status tracking** — `unsubmitted` / `submitted` / `marked` / `missing` states resolved live from Firestore
- **Light/dark theme support and responsive layout** built with Tailwind

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router) with Turbopack |
| Language | TypeScript, React 19 |
| Styling | Tailwind CSS v4 |
| Auth | Firebase Authentication (email/password) |
| Database | Cloud Firestore (NoSQL) |
| AI | OpenAI API (GPT-4.1 for marking, GPT-4o for feedback synthesis) |
| Deployment | Firebase Hosting (recommended) or AWS |

## My Contributions

This was a six person capstone, and I was responsible the server-side and data layer end-to-end. Specifically:

- **AI marking pipeline** — designed and implemented the two-stage LLM marking flow described below (`/api/markClass`): subject-aware prompt construction, rubric serialisation, JSON-only output contract, the realism guard that deflates inflated scores, optional cohort z-score normalisation, and the `AIProvider` abstraction that makes the underlying model swappable between OpenAI, Anthropic, Gemini, and self-hosted models
- **Backend / API layer** — Next.js App Router route handlers, request validation, error handling, and the batching logic that processes up to 30 student submissions per call
- **Database (Firestore) CRUD** — designed the data model (`teacher`, `student`, `assignment`, `rubric`, `completedAssignment`, `studentGrade`), wrote all read/write queries across teacher and student flows, and used deterministic composite doc IDs (`${studentId}_${assignmentId}`) so re-marking is idempotent
- **Authentication & role-based routing** — Firebase Auth email/password integration, post-login Firestore role lookup, and the routing logic that sends teachers and students to their respective dashboards

The frontend pages, rubric and assignment builder UI, and client-facing documentation were owned by other team members.

## AI Marking Architecture

The marking endpoint lives at `POST /api/markClass`. It accepts an `assignment` (with attached rubric) and an array of `submissions`, then runs a **two-stage LLM pipeline** for each student.

### Stage 1 — Per-question rubric scoring (GPT-4.1)

For each submission (capped at 30 per request to keep latency bounded and stay within rate limits), the route iterates through every question and:

1. Matches the rubric block for that question (by index or normalised question text)
2. Computes `maxScore` as the sum of criterion mark allocations
3. Builds a **subject-aware system prompt** — the marking focus shifts depending on whether the subject is English (clarity, structure, expression), Maths (numeric accuracy, working shown), Science (factual accuracy, scientific vocabulary), History (cause-and-effect, evidence) or Art (creativity, technique, reflection)
4. Frames the model as an NSW Stage 2–3 primary teacher with explicit grade-band rubric (Excellent / Good / Satisfactory / Developing / Beginning) so scores stay realistic and age-appropriate
5. Calls the LLM with `temperature: 0.15`, low `presence_penalty`, and high `frequency_penalty` for stable, consistent scoring
6. Parses a JSON response of `{ criteria, score, comment }`, clamps the score to `[0, maxScore]`

A **realism guard** then runs on the parsed output: if the awarded score is above 80% but the comment contains two or more negative phrases (`lack`, `missing`, `needs`, `improve`, `weak`, `unclear`, `incomplete`, `error`, `issue`, …), the score is deflated by 20% (floored at 60% of max) to keep numeric scores aligned with the written justification.

### Stage 2 — Holistic feedback synthesis (GPT-4o)

After per-question marking finishes for a student, a second LLM call takes:

- Every per-question comment and score
- The overall percentage
- A response-length tier (extended writing / short response / simple task) computed from total answer word count

…and asks GPT-4o to write a short, friendly, teacher-style feedback paragraph that starts with a positive, offers one gentle improvement suggestion, briefly addresses grammar and sentence structure, and uses "I" statements to sound authentic. Output length is adapted to task length (1–2 sentences for short tasks, 3–5 for extended writing).

If either LLM call fails or returns malformed JSON, the route falls back to a deterministic, percentage-banded feedback string so a student never sees a broken result.

### Provider abstraction

All LLM calls go through a single `AIProvider` interface with a `chatCompletion()` method, fetched via a `getAIProvider()` factory. This means the underlying model can be swapped between OpenAI, Anthropic Claude, Google Gemini, AWS Bedrock, or a self-hosted model without touching any of the marking logic.

### Optional cohort normalisation

A `NORMALIZE_MARKS=true` environment flag enables post-hoc z-score normalisation: if more than three submissions are marked and the standard deviation of grades is below 10 (i.e. the LLM has clustered everyone too tightly), grades are re-mapped onto a distribution centred on 70 with σ = 15 to restore spread. Off by default.

### Persistence

Results are written back to the `studentGrade` Firestore collection with deterministic doc IDs (`${studentId}_${assignmentId}`), so re-marking is idempotent and grades remain linked to their assignment and rubric.

## Project Structure

```
src/app/
├── api/markClass/route.ts        # Two-stage AI marking endpoint
├── page.tsx                      # Login (Firebase Auth + role routing)
├── firebase.ts                   # Firebase client init
├── teacher-dashboard/            # Teacher home
├── teacher-assignments/          # Create / manage assignments
├── teacher-rubrics/              # Per-question rubric builder
├── teacher-grades/               # Review and edit AI-marked grades
├── teacher-past-assignments/     # Archive view
├── student-dashboard/            # Student home
├── student-assignments/          # Submit answers
├── student-grades/               # View grades and feedback
├── classes/[id]/                 # Class detail + batch marking trigger
└── settings/                     # User settings
```

## Getting Started

### Prerequisites

- Node.js (LTS) and npm
- A Firebase project with Auth (email/password) and Firestore enabled
- An OpenAI API key (or any compatible provider, via the `AIProvider` interface)

### Setup

```bash
npm install
```

Create `.env.local` in the project root:

```env
NEXT_PUBLIC_FIREBASE_API_KEY=...
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=...
NEXT_PUBLIC_FIREBASE_PROJECT_ID=...
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=...
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=...
NEXT_PUBLIC_FIREBASE_APP_ID=...
NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID=...

OPENAI_API_KEY=...
NORMALIZE_MARKS=false
```

### Run

```bash
npm run dev
```

Then open [http://localhost:3000](http://localhost:3000).

Convenience launchers are included: `run.bat` (Windows) and `start.command` (macOS/Linux) handle `npm install` and `npm run dev` in one step.

## Firestore Data Model

| Collection | Document shape |
|---|---|
| `teacher` | Teacher profile keyed by Firebase Auth UID |
| `student` | Student profile keyed by Firebase Auth UID, with `class` |
| `assignment` | Questions, subject, class, due date |
| `rubric` | Per-question criteria with title, description, marks |
| `completedAssignment` | Student submission with answers indexed by question |
| `studentGrade` | AI-generated grade, feedback, criteria breakdown, editable by teacher |

Delivered for the UTS Software Development Studio program.

## License

This project was developed under a client engagement. Reuse or access to source code is strictly prohibited.
