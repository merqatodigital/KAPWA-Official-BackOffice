# KAPWA Hospitality OS

> A self-hosted, boutique-resort operations platform. One property, one private
> Supabase project, one domain — no multi-tenant SaaS.

**KAPWA Hospitality OS** ("KAPWA OS") is a private, per-customer deployment that
runs the full day-to-day operations of a small resort or boutique hotel: front
desk, housekeeping, kitchen, bar, waitstaff, cashier, tours/experiences, and a
branded guest portal — plus an admin back office with reporting, staff/RBAC,
billing, and audit logging. An optional self-hosted voice concierge ("TALA")
and an AI assistant ("Hermes") layer on top.

Live demo: https://kapwa-resort-webapp.vercel.app

---

## Why "KAPWA"?

"Kapwa" is a Filipino/Tagalog concept of shared identity — *treat the other as
yourself*. The product framing puts staff and guests on the same operational
surface: a single source of truth for who is arriving, what needs cleaning,
what was ordered, and what was paid.

---

## Features

### Customer-facing surfaces (by role)

| Module | What it does |
| --- | --- |
| **Front desk / Reception** | Arrivals, departures, room status, guest search, room billing, cashier flows. |
| **Housekeeping** | Inspection workflow, unit status changes, task queue, housekeeping config. |
| **Kitchen** | Department order board; item prep/status workflow. |
| **Bar** | Department order board + menu dependency flows. |
| **Waitstaff** | Order type → menu → cart/drawer → department routing. |
| **Cashier / Payments** | Room billing modal, payment recording, checkout flow. |
| **Tours / Experiences** | Booking config, availability calendar, pricing/slots, "today" view. |
| **Guest portal** | Room-branded guest view (dynamic resort-name fallback). |
| **Service boards** | Live board views by department. |
| **Reports** | P&L report/HTML generation, visual breakdown charts, print-to-PDF export. |
| **OCR** | Receipt scanning via `scan-receipt` Edge Function → AI gateway. |
| **Staff portal** | Role-gated shells + permission checks, PIN login (mints staff JWT). |

### Admin back office

- Configuration, reporting, staff management, billing, expense/budget views.
- Audit logging.
- Bot/AI settings (`/admin/bot-settings`).
- Company onboarding: roles, departments, menus, services, pricing, tour slots.

### Intelligence layer (optional)

- **Hermes assistant** — daily owner summary, guest-response drafting, staff
  task assistance, review/marketing support, with human-approval controls.
  Runs as Supabase Edge Functions (`admin-summary`, `ops-coordinator`,
  `concierge-ai`, `reservations-ai`, …).
- **TALA voice concierge** — see [`voice-agent/README.md`](voice-agent/README.md).
  Self-hosted LiveKit + Ollama/Qwen3 + whisper.cpp + Kokoro. Zero per-call
  cost because everything runs on your own hardware.

---

## Architecture

```
Browser (React PWA)
  ├─ UI, offline shells, local form state, print-to-PDF
  └─ Supabase (auth, Postgres, storage, Edge Functions, webhooks)
        ├─ Edge Functions: admin-summary, concierge-ai, employee-auth,
        │   forecast-7day, frontdesk-today, guest-requests-api, guest-search,
        │   housekeeping, ops-coordinator, orders-today, process-webhook-queue,
        │   reservations-ai, send-telegram, sirvoy-webhook, today-ops,
        │   tours-today, scan-receipt
        └─ Optional: Hermes / Ollama / LiveKit stack (voice concierge,
            daily summary, agent loops) — private, customer-controlled.
```

**Data isolation:** one Supabase project per customer; one service-role secret
known only to the deployment and Edge Functions; staff credentials and guest
data never cross customers.

---

## Tech stack

- **Frontend:** Vite + React 18 + TypeScript, shadcn/ui (Radix), Tailwind CSS
- **State/data:** TanStack Query, Zustand, react-hook-form + zod
- **Backend:** Supabase (Postgres + Row Level Security, Auth, Storage,
  Edge Functions via Deno/TypeScript)
- **AI:** OpenRouter/Claude-based staff triage (`concierge-ai`) and a
  self-hosted Ollama/Qwen3 stack for TALA
- **Voice (optional):** LiveKit + whisper.cpp (STT) + Kokoro (TTS)
- **Charts/PDF:** Recharts, jsPDF
- **Tooling:** ESLint, Vitest, Bun (lockfile), Vercel (deploy)

---

## Project structure

```
kapwa-resort-webapp/
├─ src/                  # React app (pages, components, hooks, lib, integrations)
│  ├─ pages/             # Index, Reception, Kitchen, Bar, Housekeeper,
│  │                     #   GuestPortal, Admin, Experiences, Service* shells…
│  ├─ components/        # UI + role guards (RequireAuth)
│  └─ integrations/      # Supabase client + types
├─ supabase/
│  ├─ functions/         # Edge Functions (Deno/TS)
│  └─ migrations/        # DB schema migrations
├─ voice-agent/          # TALA self-hosted voice concierge (Python + PWA)
├─ server/               # Optional local Hermes proxy
├─ docs/                 # Pilot plan, security notes
├─ public/               # Static assets
└─ vercel.json           # Deploy config
```

See [`docs/KAPWA-OS-COMMERCIAL-PILOT-PLAN.md`](docs/KAPWA-OS-COMMERCIAL-PILOT-PLAN.md)
for the scope, packages, and install checklist.

---

## Getting started

### Prerequisites

- Node.js 22+ (or Bun)
- A Supabase project
- (Optional) Docker + a mic for the TALA voice agent

### 1. Install

```sh
git clone <YOUR_GIT_URL>
cd kapwa-resort-webapp
npm install        # or: bun install
```

### 2. Configure environment

```sh
cp .env.example .env
```

Fill in your Supabase project values:

```env
VITE_SUPABASE_PROJECT_ID="your-project-ref"
VITE_SUPABASE_URL="https://your-project-ref.supabase.co"
VITE_SUPABASE_PUBLISHABLE_KEY="your-anon-key"
VITE_USE_STAFF_JWT="false"          # enable after STAFF_JWT_SECRET + RLS migration
VITE_INTERNAL_FN_SECRET="change-me" # also set in Supabase → Edge Functions → Secrets
```

> Edge-function secrets (`INTERNAL_FN_SECRET`, `STAFF_JWT_SECRET`,
> `OPENROUTER_API_KEY`, `LOVABLE_API_KEY`, `TELEGRAM_BOT_TOKEN`) are set in
> Supabase, **not** in `.env`. See `.env.example` for the full list.

### 3. Apply database migrations

```sh
supabase db push        # or paste supabase/migrations/*.sql into the SQL editor
```

### 4. Run locally

```sh
npm run dev             # frontend (Vite)
npm run dev:server      # optional local Hermes proxy (server/)
```

### 5. Test / lint / build

```sh
npm run test            # Vitest
npm run lint            # ESLint
npm run build           # production build
```

---

## Deploy

Frontend builds are deployed to Vercel (or any static host). The frontend build
env must contain **only** public Supabase anon settings; the internal Edge
Function secret and Hermes host/token must **never** reach the browser.

- Set the custom domain / CORS origin per deployment.
- Deploy Supabase Edge Functions: `supabase functions deploy`.

---

## Voice concierge (TALA)

`voice-agent/` is a separate, self-hosted module: LiveKit room transport +
Ollama/Qwen3 8B (LLM) + whisper.cpp (STT) + Kokoro (TTS) + Supabase (live ops
data). 12 tools query real KAPWA OS tables (availability, tasks, maintenance,
housekeeping, inventory, weather, events, FAQ…).

```sh
cd voice-agent
docker compose up --build
```

See [`voice-agent/README.md`](voice-agent/README.md) for setup, tools, and the
four agent loops (Planner → Execution → Verification → Repair).

---

## Security notes

- The repo's root `.env` has historically been committed — **rotate any secret
  that was ever in it** before going live, and scrub it from git history.
- Enable Row Level Security and the staff-JWT flow (see
  `docs/security/rls-migration.md`) before any commercial deployment.
- Remove client-side references to `VITE_INTERNAL_FN_SECRET` / internal request
  headers; move those calls into Edge Functions.
- Run `npm audit` and patch moderate/high/critical issues per customer deployment.

---

## Documentation

- [`docs/KAPWA-OS-COMMERCIAL-PILOT-PLAN.md`](docs/KAPWA-OS-COMMERCIAL-PILOT-PLAN.md) — product inventory, pilot scope, packages, install checklist
- [`docs/security/`](docs/security) — RLS migration and security status
- [`voice-agent/README.md`](voice-agent/README.md) — TALA voice agent

---

## License

See repository for license terms. Private, per-customer deployment — not offered
as public multi-tenant SaaS.
