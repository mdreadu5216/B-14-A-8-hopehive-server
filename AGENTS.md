# AGENTS.md

Express 5 + MongoDB REST API for HopeHive (TypeScript, ESM, no framework). The Next.js client lives in a separate repo.

## Commands

- `npm run dev` — `tsx watch src/index.ts`, serves on port 5000
- `npm run typecheck` — `tsc --noEmit`; this repo has **no tests, no linter, no CI**. Typecheck is the only automated verification, so run it after every change.
- `npm run build` → `dist/`, then `npm start` runs `dist/index.js`
- `npm run seed` — inserts 8 sample events

## Env

- `MONGODB_URI` and `AUTH_BASE_URL` are required; `src/config.ts` throws at import time if either is missing. That module is transitively imported by everything, including `src/seed.ts`, so even seeding fails without `AUTH_BASE_URL` set.
- `.env` is loaded automatically via `dotenv/config` in `config.ts`. Read config only from `config`, never `process.env` directly.
- `CLIENT_URL` is a comma-separated CORS allowlist; trailing slashes are stripped from each entry and from `AUTH_BASE_URL`. Do not hardcode origins.
- Client must be running at `AUTH_BASE_URL` for the server to fetch `<AUTH_BASE_URL>/api/auth/jwks`.

## ESM / TypeScript rules

`"type": "module"` + `moduleResolution: NodeNext`: **relative imports must end in `.js`** (`import { config } from "./config.js"`) even though the file is `.ts`. `tsx` tolerates a missing extension, so `npm run dev` can work while `npm run build` fails — always check with `npm run typecheck`.

## Auth

- `src/middleware/auth.ts` verifies BetterAuth JWTs against the remote JWKS (`jose`). No shared secret, no local user table.
- Caller identity (email, name) comes **only** from the verified token (`req.user`). Never accept `creatorEmail`/`userEmail` from a request body or query.
- Ownership is enforced by comparing `event.creatorEmail` with `req.user.email` → `403`. Email is normalized to trimmed lowercase in the token payload.

## Data model gotchas

- `joinedEvents` stores a **copy** of the event (title, thumbnail, eventType, location, eventDate) so lists need no join. Any event update must also `updateMany` the copies, and deleting an event must delete its join records (both already done in `src/routes/events.ts`).
- Joining is deduplicated by a unique index on `(eventId, userEmail)` in MongoDB, not by a pre-check. Catch duplicate key (`isDuplicateKeyError`, code 11000) and return `409`; do not replace this with a find-then-insert.
- Indexes are created in `connectDb()`, so the connection needs index-creation permission.
- `events` and `joinedEvents` are module-level `let` bindings assigned by `connectDb()`. Routes import them directly, so nothing may touch a collection before `connectDb()` runs.

## Dates

- Event dates are `YYYY-MM-DD` strings compared lexicographically — not `Date` objects. Keep that representation end to end.
- "Today" comes from `todayString()` in `src/dates.ts` using `APP_TIMEZONE` (default `Asia/Dhaka`), never `new Date().toISOString()`.
- Upcoming = `eventDate >= today`; creating/moving an event requires `eventDate > today`. Editing an event without changing its date is allowed even if the date is past.
- Public `GET /api/events` filters to upcoming in the query; `GET /api/events/:id` returns past events too.

## Conventions

- Routers are exported from `src/routes/*.ts` and mounted on `/api` in `src/index.ts`; route paths inside the router include the full path (e.g. `/events/:id/join`).
- All errors are JSON `{ message }`, validation errors add `{ errors: { field: message } }` via `formatIssues`. Keep messages user-facing and friendly; match the existing shape the client parses.
- Validate bodies with the zod schema in `src/validation.ts`; new event types go in `EVENT_TYPES` (validated against in list-filter queries too).
- Express 5 forwards rejected promises to the final error middleware — just `throw`/let async handlers reject instead of wrapping in try/catch, and keep the error handler last.
- Deploy target is Render (`render.yaml`): build `npm install && npm run build`, start `npm start`, health check `/`. `PORT` is injected there.
