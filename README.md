# CopyZero - Academic Integrity Platform

A web-based platform for academic assignments and timed assessments with AI-powered evaluation, AI question generation, and live proctoring.

## Overview

CopyZero provides a dual-role system for professors and students, covering two distinct workflows:

- **Assignments** — file/text submissions with AI plagiarism, AI-text, and rubric-criteria evaluation.
- **Assessments** — timed MCQ + coding exams with in-browser code execution, AI question generation, live webcam/screen proctoring, and an automated integrity score.

Students join either via a **cryptographically random 6-character code** the professor shares — nothing is visible until you've explicitly joined.

**Key Features:**
- Role-based access (Professor/Student), VIT email-domain restricted
- Join-code enrollment for both assignments and assessments
- AI evaluation of submissions (plagiarism, AI-text, rubric scoring) in a single pass — **BYOK** (bring your own Groq key)
- **Assessments**: single-timer sessions mixing MCQ and coding questions, one attempt per student
- **In-browser code execution** (Python via Pyodide, JavaScript via sandboxed Web Workers) — free, no server-side sandbox, no third-party judge
- **AI question generation** for assessments (MCQ + coding), with a mandatory professor verification step for AI-authored coding test cases
- **AI proctoring** — client-side face presence/count detection + rolling screen-share capture, with event-triggered evidence
- **Integrity score** per submission, combining behavioral + proctoring + content signals
- Professor results dashboard with scores, integrity breakdown, and proctoring timeline/evidence

## Technology Stack

**Frontend:**
- React.js + Vite
- React Router
- Tailwind CSS
- Firebase Authentication
- Pyodide (CDN) for in-browser Python; sandboxed Web Workers for JavaScript
- face-api.js (TensorFlow.js, client-side) for proctoring face detection

**Backend:**
- Node.js with Express
- Firebase Admin SDK (Auth + Firestore)
- **Groq** (llama-3.1-8b-instant) as the primary AI provider, with an NVIDIA NIM (DeepSeek V4 Flash) + HuggingFace fallback gateway for question generation

## Evaluation System

AI evaluation of a submission runs as a **single Groq call** covering:

1. **Student-to-student plagiarism** — compares the submission text against other submissions for the same assignment
2. **AI-generated text likelihood** — flags content that reads as AI-assisted
3. **Rubric criteria scoring** — scores the submission against the professor's rubric with reasoning per criterion
4. For coding submissions: a **test-result plausibility** check — since code runs client-side, the model sanity-checks whether the code logic could plausibly produce the claimed pass rate

**Final Score Calculation:**
- Plagiarism Score = MIN(student-plagiarism, AI-detection)
- Final Grade = (Plagiarism × Weight%) + (Content Quality × Weight%)

The AI evaluation (`POST /api/professor/ollama-evaluate`) returns suggested scores for the professor to review; the professor then confirms via `POST /api/professor/evaluate`, which is what actually persists the score. Evaluation is **BYOK only** — each professor supplies their own Groq key via "Configure AI" in the sidebar (stored in `sessionStorage` for that tab, never sent to or stored on the server except to make the AI call).

## Project Structure

```
academic-integrity/
├── frontend/
│   ├── src/
│   │   ├── pages/
│   │   │   ├── professor/    # Professor dashboard, assignments, evaluation
│   │   │   └── student/       # Student dashboard, submission, view scores
│   │   ├── components/        # Reusable UI components
│   │   ├── services/          # API communication
│   │   └── context/           # Authentication context
│   └── public/
├── backend/
│   ├── src/
│   │   ├── controllers/       # Request handlers (assignments, assessments, events, evidence, generation)
│   │   ├── services/          # Business logic (Groq eval, question generation, grading, integrity, Firestore)
│   │   ├── routes/            # API routes
│   │   ├── middleware/        # Auth, validation
│   │   └── utils/             # Helper functions
│   └── .env                   # Environment variables (not in repo)
└── firestore.rules            # Firestore security rules (deny-all; app is backend-only)
```

## Local Setup

### Prerequisites

- Node.js (v18+, native `fetch` is required)
- npm
- A Firebase project with Authentication (Email/Password) and Firestore enabled
- A **Groq API key** ([console.groq.com/keys](https://console.groq.com/keys), free) — professors also enter their own key in-app via "Configure AI"; the platform key below is only used as a fallback for AI question generation
- (Optional) An NVIDIA NIM API key ([build.nvidia.com](https://build.nvidia.com)) — used only as a secondary fallback for question generation
- (One-time, for proctoring) face-api.js model weights in `frontend/public/models/` — see `frontend/public/models/README.md`

### 1. Clone and install

```bash
git clone https://github.com/Ajay-1011-git/CopyZero.git
cd CopyZero
cd backend && npm install
cd ../frontend && npm install
```

### 2. Configure environment variables

Both `backend/` and `frontend/` need their own `.env` file. Copy the example and fill in real values — **never commit `.env`, it's gitignored on purpose.**

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

**`backend/.env`:**
```env
PORT=5000
NODE_ENV=development
FRONTEND_URLS=http://localhost:5173

# Firebase Web API key (Project Settings > General > Web API Key) — used
# server-side to verify passwords via the Identity Toolkit REST API
FIREBASE_WEB_API_KEY=

# Firebase Admin credentials (Project Settings > Service Accounts > Generate new key)
FIREBASE_PROJECT_ID=
FIREBASE_CLIENT_EMAIL=
FIREBASE_PRIVATE_KEY=

# Groq — primary AI provider. Used as the platform-key fallback for AI
# question generation. (Submission evaluation is BYOK: each professor
# supplies their own Groq key in-app, so evaluation does not depend on this.)
GROQ_API_KEY=
GROQ_BASE_URL=https://api.groq.com/openai/v1
GROQ_MODEL=llama-3.1-8b-instant

# Optional secondary fallbacks for question generation only
NVIDIA_NIM_API_KEY=
NVIDIA_NIM_BASE_URL=https://integrate.api.nvidia.com/v1
NVIDIA_NIM_MODEL=deepseek-ai/deepseek-v4-flash
HUGGINGFACE_API_TOKEN=
```

See `backend/.env.example` for the complete list of supported variables.

`FIREBASE_PRIVATE_KEY` must keep its `\n` sequences escaped (literal backslash-n), since `.env` files can't hold real multi-line values — the app un-escapes them at startup. Wrap the whole value in double quotes.

If `FIREBASE_PROJECT_ID`/`FIREBASE_CLIENT_EMAIL`/`FIREBASE_PRIVATE_KEY` are left empty, the backend falls back to a local `backend/firebase-service-account.json` file (also gitignored) — useful if you'd rather drop in the downloaded service-account JSON than split it into env vars.

> **Port 5000 on macOS:** macOS's AirPlay Receiver often already listens on port 5000. If the backend fails to bind, change `PORT` (e.g. `5001`) and update `VITE_API_URL` below to match.

**`frontend/.env`:**
```env
VITE_API_URL=http://localhost:5000

VITE_FIREBASE_API_KEY=
VITE_FIREBASE_AUTH_DOMAIN=
VITE_FIREBASE_PROJECT_ID=
VITE_FIREBASE_STORAGE_BUCKET=
VITE_FIREBASE_MESSAGING_SENDER_ID=
VITE_FIREBASE_APP_ID=
```

The `VITE_FIREBASE_*` values are your Firebase **web app** config (Project Settings > General > Your apps > SDK setup and configuration) — these are public client keys, safe to ship in a browser bundle, but still kept out of git so each environment can point at its own Firebase project.

### 3. Deploy Firestore rules (recommended)

This app only talks to Firestore through the backend's Admin SDK, so `firestore.rules` at the repo root denies all direct client access as defense-in-depth:

```bash
firebase deploy --only firestore:rules
```

### 4. Run it

```bash
# terminal 1
cd backend && npm run dev

# terminal 2
cd frontend && npm run dev
```

Backend runs on `http://localhost:5000` (or your custom `PORT`), frontend on `http://localhost:5173`. Sign up with a `@vit.ac.in` or `@vitstudent.ac.in` email (the app enforces this domain restriction) to get started.

## Production Deployment

**Backend (Render, or similar):**
- Set `NODE_ENV=production` in the platform's environment variables — the backend gates stack-trace exposure in error responses on this, and it must be exactly `production`, not left unset.
- Set `FRONTEND_URLS` to the deployed frontend's exact origin (e.g. `https://your-app.vercel.app`) once you know it. Requests from origins not in this comma-separated allowlist are rejected by CORS. Multiple origins can be comma-separated if you have more than one frontend deployment (e.g. a preview URL and a production URL).
- Set `FIREBASE_PROJECT_ID` / `FIREBASE_CLIENT_EMAIL` / `FIREBASE_PRIVATE_KEY` / `FIREBASE_WEB_API_KEY` and whichever AI provider keys you're using (`GROQ_API_KEY` at minimum) — see `backend/.env.example`.

**Frontend:**
- `frontend/.env.production` is committed (unlike `frontend/.env`) and is loaded automatically by `npm run build` in production mode. It currently points `VITE_API_URL` at the deployed backend. Update it if the backend URL changes.
- The Firebase web config and backend URL in that file are public client-side values by design — visible in the deployed bundle's devtools regardless of whether they're in the repo — so committing them just means `npm run build` produces a working bundle on any hosting platform without extra dashboard configuration.
- Most static-site platforms (Vercel, Netlify, Render static sites) just need the build command `npm run build` and publish directory `dist` pointed at `frontend/`.

## User Workflow

### Professor Flow

**Assignments**
1. Login with VIT email (@vit.ac.in)
2. Create an assignment (essay or code type) with rubric criteria and plagiarism/content weightages — a join code is generated automatically
3. Share the join code; students enroll with it
4. Once students submit, auto-evaluate with AI (BYOK Groq) or grade manually, override scores if needed

**Assessments**
1. Create an assessment: set a single duration timer, then add MCQ and/or coding questions — manually or via **"Generate with AI"**
2. For AI-generated coding questions, run a known-good solution against the generated test cases in the built-in verify panel (AI-authored test cases can be wrong); a question can't be added until verified or explicitly overridden
3. Publish (publishing is blocked while any AI coding question is still unverified) and share the assessment join code
4. **View Results** — per student: MCQ/coding/total scores, integrity score + signal breakdown, and the full proctoring timeline with webcam/screen evidence

### Student Flow

**Assignments**
1. Login with VIT email (@vitstudent.ac.in)
2. Join an assignment with its code, then submit a file/text response (or solve a coding question in-browser with instant sample-test feedback)
3. View evaluation results, feedback, and scores

**Assessments**
1. Join an assessment with its code, then click **Start** — grant the webcam + screen-share proctoring permissions once (a consent notice is shown)
2. Work through the MCQ and coding sections under one shared timer; run coding solutions against sample tests before submitting
3. Submit once (one attempt per student, enforced server-side by the duration + attempt limit); scores are computed immediately

## API Endpoints

**Authentication:**
- POST `/api/auth/signup` — Register new user
- POST `/api/auth/login` — User login (server-side password verification)
- GET `/api/auth/profile` — Get user profile

**Professor — Assignments:**
- POST `/api/professor/assignments` — Create assignment (generates a join code)
- POST `/api/professor/rubrics` — Create rubric
- POST `/api/professor/coding-questions` — Attach a coding question to an assignment
- GET `/api/professor/submissions/assignment/:id` — Get submissions
- POST `/api/professor/ollama-evaluate` — AI evaluation (BYOK Groq)
- POST `/api/professor/evaluate` — Persist evaluation / override

**Professor — Assessments:**
- POST `/api/professor/assessments` — Create assessment shell
- PUT `/api/professor/assessments/:id` — Update (add/edit MCQ + coding questions)
- POST `/api/professor/assessments/:id/publish` — Publish (blocks on unverified AI coding questions)
- GET `/api/professor/assessments/:id/submissions` — Results: scores + integrity + proctoring per student
- POST `/api/professor/generate-assessment-questions` — AI question generation (review only, not auto-saved)

**Student — Assignments:**
- POST `/api/student/join` — Join an assignment by code
- GET `/api/student/assignments` — Get joined assignments
- POST `/api/student/submit` — Submit assignment
- POST `/api/student/submit-code` — Submit a coding-question solution
- GET `/api/student/scores/assignment/:id` — View score

**Student — Assessments:**
- POST `/api/student/assessments/join` — Join an assessment by code
- GET `/api/student/assessments` — Get joined assessments (answers/hidden outputs redacted)
- POST `/api/student/assessments/:id/start` — Start the single attempt
- POST `/api/student/assessments/:id/submit` — Submit (server-enforced timer + one attempt)

**Proctoring & Integrity:**
- POST `/api/events/batch` — Log behavioral/proctoring events (student)
- POST `/api/events/evidence` — Upload webcam snapshot / screen clip (student, size + mime capped)
- GET `/api/events/:submissionId` — Event timeline (professor, ownership-checked)
- GET `/api/proctor/evidence/:eventId` — Evidence for an event (professor, ownership-checked)
- GET `/api/integrity/:submissionId` — Integrity score for an assignment submission

**AI key:**
- POST `/api/ai/test-key` — Validate a user-provided Groq key (rate-limited)

## AI Question Generation (Assessments)

- **"Generate with AI"** on the assessment builder produces MCQ and/or coding questions for **review** — nothing is auto-saved. The professor edits inline, then explicitly adds each question.
- **Provider chain** (separate from evaluation): Groq (professor's own key first, then the platform key with per-minute rate limiting) → NVIDIA NIM → HuggingFace fallback. Generation and evaluation can run on different providers.
- **Strict validation** server-side: MCQs must have exactly 4 distinct options and a valid 0–3 correct index; coding questions must have 2–8 test cases with at least one hidden and one visible. Malformed items are retried once, then dropped (the professor is told how many).
- **Mandatory verification for AI coding questions**: because the model authors its own test cases (and can get the expected outputs wrong), each AI coding question is flagged `aiGenerated`/unverified. The professor runs a known-good solution against the generated test cases in-panel; failing tests show `expected` vs `actual` so the professor can fix a bad expected output and re-run. A question can only be added once verified (a passing solution) or explicitly overridden — and **publishing is blocked** while any AI coding question remains unverified.
- Rate limit: 8 generations per professor per hour.

## Integrity Score

Every assessment (and evaluated assignment) submission gets a 0–100 integrity score combining behavioral signals (tab switches, focus loss, copy/paste, idle time), proctoring signals (no-face / multiple-face detections, screen-share interruptions), and content signals (plagiarism, AI-text, coding test-result plausibility). Computed once per submission and shown to the professor with a plain-English explanation and a per-signal breakdown.

## Coding & AI Proctoring

- **In-browser code execution**: Python (via Pyodide, loaded lazily from the jsDelivr CDN) and JavaScript both run in sandboxed Web Workers — never on the main thread, never server-side. Students get instant pass/fail feedback against visible test cases; hidden test cases run too but their expected output is never sent to the browser, so results can't be hardcoded.
- **Client-reported test results are a signal, not a verdict**: since execution is client-side, a submission's claimed pass rate is stored as `pending_verification` and cross-checked by the AI evaluation step (`testResultPlausibility`) — a professor should not treat 100% client-claimed pass rate on hidden tests as ground truth without that check.
- **AI proctoring (webcam + screen)**: presence/count face detection only (`face-api.js`, tiny face detector, entirely client-side, no facial recognition or identity matching) and a rolling screen-share buffer. Evidence (snapshots/clips) is captured only on flagged moments — no face detected, multiple faces, tab switch, fullscreen exit, or screen-share stopped — never continuously.
- **Data minimization**: webcam snapshots and screen clips are only ever the flagged evidence, capped at 500KB each. **These should be deleted once grading is finalized for a given assignment** — they're proctoring evidence for grading disputes, not a long-term record, and count as sensitive biometric-adjacent data. There's currently no automatic deletion job; this is a manual/cron follow-up if this app is used beyond a demo.
- Requires a one-time manual step: download `face-api.js`'s tiny-face-detector model weights into `frontend/public/models/` — see `frontend/public/models/README.md`.

## Security

- VIT email domain verification (@vit.ac.in, @vitstudent.ac.in) on signup and login
- Real password verification on login via Firebase Identity Toolkit (no user-enumeration hints — generic 401 on any failure)
- Role-based access control, enforced server-side on every route
- Ownership checks on all professor-scoped resources (assignments, rubrics, scores)
- CORS origin allowlist (localhost permitted in dev only), `helmet`, rate limiting (strict auth limiter in production, plus per-user limiters on join, code submission, AI evaluation, key testing, and generation)
- Firestore security rules deny all direct client access — the app only talks to Firestore through the backend's Admin SDK
- Secrets (Firebase Admin credentials, Groq/NIM/HuggingFace keys) loaded from environment variables, never committed; user-provided Groq keys live only in the browser tab's `sessionStorage`, never persisted server-side
- Mass-assignment protection (field allowlists) on updatable resources; ownership checks traced through the full chain (event → submission → assignment/assessment → professor)
- Coding execution runs only in sandboxed Web Workers with network/storage globals stripped — student code can't exfiltrate data or reach the page
- Proctoring evidence is size- and mimetype-capped server-side; face detection is presence/count only (no identity matching)
- Error responses are generic to clients; full errors are logged server-side only

## Known Limitations

- **AI-generated test cases can be wrong** — the mandatory professor verification step exists precisely because the model isn't reliable at computing exact expected outputs
- Client-side code execution means test results are self-reported; hidden-test results are cross-checked by the AI plausibility signal, not independently re-run server-side
- AI detection accuracy depends on the underlying model and isn't guaranteed
- Free-tier AI providers (Groq/NIM) have rate limits
- Proctoring evidence has no automatic deletion job yet — it should be purged after grading is finalized (see Data Minimization above)
- face-api.js pulls in an older TensorFlow.js dependency chain that shows an `npm audit` advisory; it is a Node-only issue that does not execute in the browser

## Contributing

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Open a Pull Request

## License

This project is licensed under the MIT License.

## Contact

For questions or issues, please open an issue on GitHub.