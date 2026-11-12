# KAARYA — Industrial Skill Verification

Candidates prove practical CNC-turning skill in a simulated machine shop; employers hire on verified evidence instead of resume claims.

This repo is the complete site: the original KAARYA front end plus a Node.js + SQLite backend.

## What the backend adds

| Area | What it does |
| --- | --- |
| **Accounts** | Candidate and employer (MSME) registration/login. Passwords hashed with scrypt, sessions in an HttpOnly cookie (only a hash of the token is stored). |
| **Server-side scoring** | The browser sends the machine configuration; the server validates it, scores it and stores the result. Scores can't be tampered with from dev tools. |
| **Two-attempt flow** | Attempt 1 → (if there's a gap) targeted knowledge check → bonus points → manually configured Attempt 2. The server enforces the order. |
| **Knowledge-check bank** | 15 questions across the five capabilities. Each quiz targets the capabilities the candidate failed to demonstrate. Correct answers are never sent to the browser before submission. |
| **Employer portal** | Ranked candidate pool, per-candidate reports with replay data, **Shortlist** and **Request shop-floor test** actions saved per employer. |
| **Jobs & partners** | Matched opportunities and partner enterprises served from the database. |
| **Hardening** | Input validation, same-origin checks on writes, rate limiting, CSP and other security headers, output escaping in the UI. |

## Run locally

Requires **Node.js 22.13+** (uses the built-in `node:sqlite`, so there is **nothing to `npm install`**).

```bash
npm start
# → http://localhost:3000
```

In development the app seeds three demo candidates (Arun, Kiran, Meena) and a demo employer:

```
demo@kaarya.dev  /  kaarya-demo-1
```

Run the tests (engine, API, auth, role isolation, rate limiting):

```bash
npm test
```

## Project layout

```
server.js            HTTP server: static files, security headers, API dispatch
src/
  api.js             All API routes
  engine.js          Deterministic scoring engine (single source of truth)
  questions.js       Knowledge-check bank + targeted quiz builder
  db.js              SQLite schema + seeding
  security.js        Password hashing, sessions, rate limiter
  http.js, config.js Helpers and environment config
public/
  index.html         App shell
  css/styles.css
  js/app.js          UI, views and API client
  js/machine.js      3D machine (three.js) + procedural audio
  js/boot.js         Session restore + startup
test/                node:test suites
Dockerfile · render.yaml · .github/workflows/ci.yml
```

## API

All endpoints are JSON under `/api`. Auth is a session cookie.

| Method & path | Who | Purpose |
| --- | --- | --- |
| `GET /health` | public | Liveness check |
| `GET /jobs`, `GET /msmes` | public | Job matches, partner list (+ demo login if seeded) |
| `POST /auth/register` | public | `role: "candidate" \| "employer"` |
| `POST /auth/login`, `POST /auth/logout`, `GET /auth/me` | — | Session management |
| `PUT /candidate/profile` | candidate | Save profile (required before assessing) |
| `POST /assessments` | candidate | Submit machine config → scored result (attempt 1 or 2 decided by the server) |
| `GET /quiz` · `POST /quiz/:id/submit` | candidate | Targeted knowledge check and grading |
| `GET /employer/candidates` | employer | Ranked pool + stats |
| `GET /employer/candidates/:id` | employer | Full report incl. machine settings |
| `POST /employer/candidates/:id/actions` | employer | `{ "action": "shortlist" \| "physical" }` |

## Configuration

See [`.env.example`](.env.example). Key variables: `PORT`, `DATA_DIR`, `TRUST_PROXY`, `SEED_DEMO`, `AUTH_RATE_MAX`.

> **Before going live:** set `SEED_DEMO=false`. With it on, a demo employer login is public and shown on the employer page.

## Put it on GitHub

```bash
cd kaarya
git init
git add .
git commit -m "KAARYA: front end + backend"
git branch -M main
git remote add origin https://github.com/<your-username>/kaarya.git
git push -u origin main
```

CI (`.github/workflows/ci.yml`) runs the test suite on every push.

## Deploy

GitHub Pages can't run a backend, so deploy to a host that runs Node/Docker.

**Render** — New → Blueprint → select this repo (`render.yaml` is included). The free plan has no persistent disk, so the database resets on each deploy; for real data use a paid plan with the disk block in `render.yaml` uncommented.

**Railway / Fly.io / any Docker host** — the included `Dockerfile` works as is. Mount a persistent volume at `/data` and set `TRUST_PROXY=true`.

**Any VPS** — `NODE_ENV=production TRUST_PROXY=true node --disable-warning=ExperimentalWarning server.js` behind nginx/Caddy for HTTPS.

## Notes

- The 3D machine and fonts load from CDNs (cdnjs, Google Fonts), so those need internet access in the browser.
- SQLite is perfect for a single instance. To scale horizontally, swap `src/db.js` for Postgres.
- Rate limiting is in-memory (per process).
