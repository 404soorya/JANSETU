# AI Government Office Clerk

A citizen describes a request in plain language through chat. The clerk asks
for each missing item, stores structured answers, generates a government-style
application PDF, and routes the file to a reviewing officer.

## Current scope (MVP)

- One service type: birth certificate copy request (seeded).
- Chat with schema-driven questions. With an LLM key, free-text extraction is
  added on top of the same schema.
- Field-level correction, per-field document upload (10 MB max, white-listed
  formats).
- Status machine with full history: COLLECTING → CONFIRMING → CONFIRMED →
  GENERATING → READY_FOR_REVIEW → CITIZEN_APPROVED → SUBMITTED → UNDER_REVIEW
  → APPROVED / RETURNED / REJECTED / GENERATION_FAILED.
- Deterministic PDF generation (`@react-pdf/renderer`), versioned files.
- Citizen dashboard and officer queue with approve / return / reject.
- Email + password auth, role-based access, audit log for case actions.

## Run it

Requirements: Node.js 18+ (Node 24 tested), npm. Bun works for the seed.

```powershell
npm install
npm run db:push   # create the SQLite database from the Prisma schema
npm run db:seed   # demo users + one service type
npm run dev       # http://localhost:3000
```

Demo logins (password `password123`):

| Email | Role |
|---|---|
| citizen@example.com | Citizen |
| officer@example.com | Officer |
| admin@example.com | Admin |

## Configure

Copy `.env.example` to `.env` if it does not exist. Key variables:

| Variable | Meaning |
|---|---|
| `DATABASE_URL` | SQLite for the MVP. File path is relative to `prisma/`. |
| `SESSION_SECRET` | At least 16 characters. Signs session cookies. |
| `LLM_API_BASE` | Optional. Any OpenAI-compatible endpoint. |
| `LLM_API_KEY` | Optional. Empty key = assisted mode (no free-text parsing). |
| `LLM_MODEL` | Optional. Model name for the chat API. |

## Test

Start the server (`npm run build` then `npm run start`, or `npm run dev`), then:

```powershell
powershell -ExecutionPolicy Bypass -File scripts\smoke-test.ps1
```

The script walks the full loop: login → start case → answer questions → upload
ID proof → confirm → generate → approve → submit → officer approves → PDF
download check.

## Architecture

```
src/app/api/...        route handlers (auth, cases, messages, fields,
                       documents, confirm, generate, approve, submit,
                       decision, file download)
src/lib/case-service   every case mutation: turn processing, field storage,
                       status transitions, question selection
src/lib/status         status machine rules (single source of truth)
src/lib/service-defs   field schema shape + value validation
src/lib/llm            optional LLM adapter (classify + extract), fallback-safe
src/lib/pdf            deterministic PDF template
src/lib/storage        upload/persisted-file helpers (uploads/ dir)
src/app/               pages: landing, chat, citizen case detail, officer queue
prisma/                schema (SQLite) + seed
```

The PDF is a pure function of stored data. The LLM never writes PDF content.
When the LLM is unavailable or returns invalid values, the flow falls back to
asking each field directly; stored data is never incorrect.

## Next steps (post-MVP)

1. Second service type + admin editor for schemas.
2. Background job queue for PDF generation and notifications.
3. Multi-language chat.
4. Postgres migration (one schema line) when concurrency matters.
5. External submission gateway integration, payments, e-signatures.

## Notes

- The project lives inside a OneDrive folder. OneDrive syncing
  `node_modules/` and `.next/` is slow; both are gitignored. Consider moving
  the repo outside OneDrive for day-to-day development.
