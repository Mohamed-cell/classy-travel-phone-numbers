# Sabre Harvest — Classy OS Feature Design

**Date:** 2026-07-06
**Status:** Approved by Mohamed (brainstorm 2026-07-06); pending written-spec review
**Replaces:** the standalone pyautogui script in this repo (`main.py`)

## 1. Vision

The admin opens a **Sabre Harvest** section in Classy OS, enters everything the
scraper needs once (a dedicated Sabre login, runner pairing), then for each
harvest just picks a Sabre queue and clicks **Start**. An invisible background
process on his Windows PC works the entire queue via a headless browser, the
run finishes by itself when the queue is empty, and the result appears in
Classy OS as a neatly organized, date-stamped Excel file — plus an optional,
approval-gated import into the CRM's contacts.

## 2. Decisions locked during brainstorm

| # | Question | Decision |
|---|----------|----------|
| 1 | Sabre access path | **Browser login (Sabre Red 360 web)** — drives the web emulator headlessly; no desktop-app GUI automation |
| 2 | Sabre credential | **Dedicated agent sign-in (EPR)** just for the bot — no session conflicts with the admin |
| 3 | Where it runs | **Admin's Windows PC** via a small installed runner (not cloud) |
| 4 | Data destination | **Both**: date-stamped Excel in Classy OS **and** approval-gated CRM contact import |
| 5 | Row shape | **Enriched**: phone/email + passenger name + PNR locator + queue + scrape timestamp |
| 6 | AI involvement | **Fallback + monitoring**: gpt-4o-mini rescue-parses only failed pages; plain-language run report. Server-side only, via the OS's existing gated LLM adapter |
| 7 | Approach | **A — Node.js + Playwright headless runner**, one stack with the OS |

## 3. Architecture

Three components. The OS is the brain and the vault; the runner is a stateless
pair of hands.

### 3.1 Classy OS module `sabre-harvest` (in the classy-travel-ghl-os repo)

Follows every existing OS convention: server components → `requireContext` →
section services in `src/lib/sabre-harvest/` → account-scoped repository reads
(no drizzle outside `src/lib/db/**`), RBAC grants, audit logs, approval queue,
i18n fr/en/ar (fr canonical).

**Pages**
- `/sabre-harvest` — runs list + live status of the active run + New Run.
- `/sabre-harvest/settings` — Sabre credentials, runner pairing/revocation.

**New RBAC grants:** `sabre_harvest:view`, `sabre_harvest:edit` (settings),
`sabre_harvest:run` (start/stop), plus the CRM import routes through the
existing `approval_requests` flow (`contacts` domain import action).

### 3.2 Runner (this repo, rebuilt as `sabre-harvest-runner`)

- Node.js + TypeScript + Playwright (headless Chromium), packaged as a single
  Windows executable; installed once as a Windows service (auto-start, no
  window, no tray).
- **Outbound-only** transport: long-polls `POST /api/sabre-runner/poll` every
  3–5 s with a Bearer token. No inbound ports, no firewall config.
- Pairing: admin generates a one-time pairing code in Settings → runner
  exchanges it for a long-lived token. OS stores only the token's hash
  (mirrors the OS's Hermes `service_accounts` pattern). Revocable instantly.
- On job claim: receives job config + Sabre credentials (decrypted server-side,
  delivered over TLS, held **in memory only** — never written to the PC's disk).
- Queue loop (same logic as `main.py`, made deterministic):
  1. Sign in to Sabre Red 360 web; open the emulator.
  2. Access the queue (`Q/<n>`).
  3. Per record: send `*H9*PE*HI`, read the emulator's text from the DOM,
     extract contacts, POST the batch to the OS, send `I` to advance.
  4. Detect Sabre's queue-empty response → report `completed`.
- Heartbeat every ~15 s (piggybacked on poll/results calls). Poll responses can
  carry a `stop` signal → finish current record, report `stopped`.
- If a screen has contact-like signals but zero extraction, POST the raw screen
  text to the OS for server-side AI rescue (the OpenAI key never leaves Railway).

### 3.3 OS server-side finishing (existing Railway worker)

On run completion: generate the Excel with exceljs, upload to R2 (existing doc
storage), write the artifact row, produce the run report. CRM import (on
admin's click + approval) reuses the proven import-pipeline conventions:
deterministic uuidv5 ids, `ON CONFLICT DO NOTHING`, natural-key dedup by
normalized phone/email, merge into existing contacts rather than duplicate,
tag `import:sabre-harvest-<runId>` (+ `source:sabre`), lifecycle `lead`.

## 4. Data model (4 new tables, account-scoped)

- `sabre_credentials` — id, account_id, label, pcc, encrypted_blob
  (AES-256-GCM over `{agentId, password}`; key = env `SABRE_CRED_KEY`,
  referenced by NAME only), created_at, updated_at. One active row per account.
- `sabre_runners` — id, account_id, name, token_hash, paired_at, last_seen_at,
  revoked_at.
- `sabre_runs` — id, account_id, runner_id, queue_number, title, status
  (`queued|running|completed|failed|stopped`), records_processed,
  phones_found, emails_found, started_at, finished_at, last_heartbeat_at,
  error_reason, excel_r2_key, report_text, report_kind (`ai|deterministic`),
  partial (bool), created_by.
- `sabre_run_contacts` — id, run_id, account_id, kind (`phone|email`),
  value_normalized (10-digit NANP string for phones, lowercased email), passenger_name,
  pnr_locator, raw_line, needs_review (bool, AI-rescue queue), extracted_by
  (`regex|ai`), imported_contact_id (nullable). Unique on
  (run_id, kind, value_normalized).

## 5. API surface

**Runner-facing** (Bearer runner-token, hashed lookup, rate-limited, all writes
audited):
- `POST /api/sabre-runner/pair` — one-time code → token.
- `POST /api/sabre-runner/poll` — heartbeat; returns pending job (with
  credentials on first claim) and/or `stop` signal.
- `POST /api/sabre-runner/runs/:id/results` — batch of extracted rows +
  counters (idempotent via the unique key).
- `POST /api/sabre-runner/runs/:id/rescue` — raw screen text for AI fallback.
- `POST /api/sabre-runner/runs/:id/finish` — `completed|failed|stopped` +
  reason.

**Admin-facing** (session auth + grants): create/start run, stop run, list
runs, run detail (live counters via polling), download Excel (signed R2 URL),
trigger CRM import (enqueues approval), settings CRUD (credentials write-only —
never returned; UI shows only "credentials set ✓"), pairing-code mint, runner
revoke.

## 6. Extraction rules

- **Primary (deterministic, in the runner):** phones from `A9` lines via the
  existing flexible 10/11-digit regex, normalized to 10 digits; emails from
  `¥…¥` fields with plain-email fallback; passenger name parsed from the PNR
  name field (`1.1LASTNAME/FIRSTNAME TITLE` convention → "Firstname Lastname");
  PNR locator from the record header. Formats validated against a fixture
  library of captured real screens.
- **AI rescue (server-side, only on failure):** page had contact-ish signals
  (digit density / `@` present) but extraction found nothing → gpt-4o-mini via
  the OS's existing LLM adapter (`LLM_MODE` double-gate respected). Until the
  live-LLM flip, rescued pages are stored as `needs_review` rows instead.
- Dedup: within run by unique key; across CRM at import time by natural keys.

## 7. Excel artifact

Auto-titled `Sabre Harvest — Queue <n> — YYYY-MM-DD.xlsx` (suffix `— partial`
when applicable):
- **Contacts** sheet: Type, Value, Passenger Name, PNR, Queue, Scraped At.
- **Summary** sheet: run metadata + counters + report text.
- (Future, if a legacy workflow needs it: one-column PHONES/EMAILS sheets.)

## 8. Failure handling

- Runner offline → run stays `queued`, UI banner "runner offline since <t>"
  from `last_seen_at`.
- Sabre login failure / kicked session → `failed` + reason shown.
- Mid-run crash → posted batches are durable; heartbeat timeout (>90 s) marks
  the run `failed` at record N; partial Excel still generated, flagged partial.
- Runaway protection: stop on Sabre queue-empty message (primary), plus
  max-records cap and max-runtime cap (config defaults 5,000 / 4 h).
- Stop button → graceful stop after current record.
- Re-running a queue is always safe (dedup end-to-end).

## 9. Security

- Sabre password AES-256-GCM at rest; decryption key only in Railway env;
  credentials delivered per-job over TLS; memory-only on the PC.
- Runner token hashed at rest, revocable; endpoints rate-limited.
- Every start/stop/import in `audit_logs`; CRM import behind the approval
  queue (SoD — requester cannot self-approve).
- New grants keep the section invisible to unprivileged roles.
- OpenAI key stays server-side; only failed pages' raw text is sent to the LLM
  (per Mohamed's standing compliance decision of 2026-06-23).

## 10. Testing

- Extractor unit tests over the captured-screen fixture corpus (incl. odd
  formats, empty records, French text).
- Excel golden-file test; import idempotency test (re-import → 0 duplicates);
  standard isolation/permission tests for every new endpoint and repo read.
- Runner E2E against a local mock "Sabre web" page (login → queue → 3 records
  → empty message).
- Phase-0 spike is the live-fire test of the only real unknown.

## 11. Build phases

- **Phase 0 — spike (GO/NO-GO, ~half day):** scripted Playwright login to
  Sabre Red 360 web with the dedicated EPR; verify unattended login (no
  blocking 2FA/CAPTCHA), command send, emulator text read, `I`/queue flow.
  NO-GO → regroup (options: desktop-app accessibility-tree automation on the
  PC, or revisit Sabre API access).
- **Phase 1 — end-to-end harvest:** OS tables/APIs/minimal runs page; runner
  MVP launched by hand from a terminal; first real queue → real Excel in OS.
- **Phase 2 — full vision:** service install + pairing UX, live progress
  polish, safety valves, AI rescue + run report, approval-gated CRM import,
  i18n, docs.

## 12. Risks & notes

- **Sabre ToS gray zone:** automating the web terminal is the same gray zone
  the current pyautogui script occupies, done more reliably and attributably
  (dedicated EPR). Accepted by owner direction; revisit official Sabre APIs if
  Sabre ever pushes back.
- **Sabre web UI changes** can break selectors → the fixture corpus + spike
  script double as a fast re-validation harness.
- **One run at a time** (single EPR): enforced server-side; a second Start
  while one is running is rejected with a clear message.
- Out of scope for v1: scheduled/recurring runs, multi-queue batching,
  multiple runners, bare-list sheets, non-queue harvest sources.
