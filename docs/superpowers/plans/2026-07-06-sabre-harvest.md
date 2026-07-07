# Sabre Harvest Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Read first:** the approved spec at `docs/superpowers/specs/2026-07-06-sabre-harvest-classy-os-design.md` (this repo). The spec is the contract; this plan is the route.
>
> **Plan style note (deliberate):** novel logic (crypto, pairing, extraction, queue engine, Excel, import) is given as real code. OS-side boilerplate steps instead point at a named exemplar file in the classy-travel-ghl-os repo to mirror — the OS has strict, audited conventions and copying its living patterns beats frozen snippets. Executors have repo access; read the exemplar before writing.

**Goal:** Rebuild the Sabre contact scraper as "Sabre Harvest": a Classy OS admin section that commands an invisible headless-browser runner on the admin's Windows PC, harvesting phones/emails/names/PNR/destination from a Sabre queue into a date-stamped Excel + approval-gated CRM import.

**Architecture:** Three components — (A) OS module `sabre-harvest` in the classy-travel-ghl-os repo (tables, runner API, admin UI, Excel/R2, CRM import), (B) Node+Playwright runner in THIS repo (outbound-only long-poll worker), (C) rollout. Runner is stateless hands; OS is brain + vault.

**Tech Stack:** OS: Next.js 16 / Drizzle / Railway Postgres / pg-boss / R2 / exceljs (new dep). Runner: Node 22 + TypeScript + Playwright, packaged to a single Windows .exe (`bun build --compile` or `@yao-pkg/pkg`), WinSW for service install.

## Global Constraints

- **Two repos.** Runner + spike live in `/mnt/c/projects/classy-travel-phone-numbers`. OS work lives in `Mohamed-cell/classy-travel-ghl-os` — do OS work from a **fresh ext4 worktree off latest master** (e.g. `git -C /mnt/c/projects/classy-travel-ghl-os worktree add ~/ctos-sabre -b feat/sabre-harvest origin/master`), NEVER on the `/mnt/c` checkout directly (it has carried other sessions' dirty WIP branches; also DrvFs breaks drizzle-kit — see memory `wsl-mnt-c-railway-up-nul-corruption`).
- **Migration numbering is contested.** Prod was at 0000–0030 as of 2026-07-06 and an unmerged WIP `0029→0031` rename was pending. At build time run `ls drizzle/` on current master and take the next free number; grep this plan's `NNNN` placeholder accordingly.
- **OS conventions are law:** `drizzle-orm` imports only in `src/lib/db/**`; every repo read/mutator calls `requirePermission` (CI guard `tests/isolation/repo-read-gate.test.ts` will fail you otherwise); governed writes route through `approval_requests`; audit logs on state changes; env vars by NAME only + registered in `src/env.ts`; i18n fr/en/ar (fr canonical, Dictionary type derives from fr); never pass non-Server-Action functions Server→Client components.
- **Deploys and prod migrations are APPROVAL-REQUIRED — never autonomous.** Deploy flow: rsync to `/home/labib/ctos-deploy` (ext4), `railway up` from INSIDE that dir. Full gotchas in memory `project-classy-travel-ghl-os`.
- **No spending/provisioning** without explicit approval (hard rule). Nothing in this plan requires new paid services.
- **LLM usage** only through the OS's existing adapter (`src/lib/integrations/llm/`) — respects the `LLM_MODE`/`LLM_ALLOW_LIVE` double gate; feature must fully work with mock (deterministic report, rescue pages parked as `needs_review`).
- **Prereqs from Mohamed (blockers for Phase 0, not for coding):** dedicated Sabre EPR credentials + the agency's Sabre Red 360 web login URL; agreement on which queue to test with.
- Runner holds Sabre credentials **in memory only**; its own bearer token may persist on disk (restricted ACL). OpenAI key never leaves Railway.
- One run at a time per account (single EPR) — enforce server-side.

## File Structure (target)

**This repo (runner):**
```
spike/probe.ts                  Phase-0 GO/NO-GO probe (throwaway quality, kept for re-validation)
spike/FINDINGS.md               spike results: selectors, timings, GO/NO-GO
runner/package.json             runner workspace (independent of spike)
runner/src/config.ts            config load/save (os url, token, caps)
runner/src/os-client.ts         poll/results/rescue/finish HTTP client w/ retry
runner/src/sabre/session.ts     Playwright login + emulator primitives (from spike findings)
runner/src/sabre/mock-server.ts local mock Sabre web app for tests
runner/src/extract.ts           regex extraction: phones/emails/name/locator/destination
runner/src/iata-countries.json  IATA→country table (bundled, offline)
runner/src/engine.ts            queue state machine (the harvest loop)
runner/src/index.ts             CLI: pair | run-once | serve (service mode)
runner/test/*.test.ts           vitest: extract, engine-vs-mock, client retry
runner/winsw/sabre-runner.xml   WinSW service definition
```

**OS repo (branch `feat/sabre-harvest`):**
```
src/lib/db/schema/sabre-harvest.ts        4 tables
drizzle/NNNN_sabre_harvest.sql            generated migration
src/lib/sabre-harvest/crypto.ts           AES-256-GCM cred sealing
src/lib/sabre-harvest/repo.ts             gated repository (creds/runners/runs/contacts)
src/lib/sabre-harvest/service.ts          pairing, run lifecycle, stop, counters
src/lib/sabre-harvest/excel.ts            exceljs artifact builder
src/lib/sabre-harvest/report.ts           deterministic + AI run report
src/lib/sabre-harvest/import.ts           run-contacts → CRM contacts mapping (uuidv5)
src/app/api/sabre-runner/{pair,poll}/route.ts
src/app/api/sabre-runner/runs/[id]/{results,rescue,finish}/route.ts
src/app/(app)/sabre-harvest/page.tsx      runs list + live active run
src/app/(app)/sabre-harvest/settings/page.tsx
src/components/sabre-harvest/*            client islands (live counters, forms)
tests/isolation/sabre-harvest.test.ts     permission-negative tests
tests/integration/sabre-harvest-*.test.ts api + import idempotency + excel golden
```

---

## Part 0 — Phase 0 spike (GO/NO-GO, do FIRST, this repo)

### Task 0.1: Spike scaffold + probe script

**Files:** Create `spike/package.json`, `spike/probe.ts`, `spike/.gitignore` (`artifacts/`, `node_modules/`, `.env`)

**Interfaces:** Produces `spike/FINDINGS.md` + working selectors consumed by Task R3.

- [ ] **Step 1:** `cd spike && npm init -y && npm i -D typescript tsx @types/node && npm i playwright && npx playwright install chromium`
- [ ] **Step 2:** Write `spike/probe.ts` (env: `SABRE_WEB_URL`, `SABRE_USER`, `SABRE_PASS`, `SABRE_PCC`, `HEADLESS=0|1`):

```ts
import { chromium } from 'playwright';
import * as fs from 'node:fs';

const art = (n: string) => `artifacts/${Date.now()}-${n}`;
fs.mkdirSync('artifacts', { recursive: true });

async function main() {
  const browser = await chromium.launch({ headless: process.env.HEADLESS !== '0' });
  const page = await browser.newPage();
  const log = (m: string) => console.log(`[spike] ${m}`);

  await page.goto(process.env.SABRE_WEB_URL!, { waitUntil: 'networkidle' });
  await page.screenshot({ path: art('login.png') });
  // STEP A — login. Selectors unknown until first run: inspect login.png / DOM dump,
  // then fill below and re-run. Record final selectors in FINDINGS.md.
  // await page.fill('<user-selector>', process.env.SABRE_USER!); ...
  fs.writeFileSync(art('login-dom.html'), await page.content());

  // STEP B — after manual selector wiring: locate the emulator input/screen,
  // send a harmless command (e.g. QC/ or *S*), read the response text:
  // await emulatorInput.type('QC/\n');  const text = await screenArea.innerText();
  // STEP C — assert: unattended login (no 2FA/CAPTCHA interstitial), command
  // round-trip < 5s, screen text machine-readable, `I` + queue flow behaves.
  await browser.close();
}
main().catch(e => { console.error(e); process.exit(1); });
```

- [ ] **Step 3:** Run headed (`HEADLESS=0 npx tsx probe.ts`) with the dedicated EPR; iterate selectors until login + command round-trip works; then confirm the same **headless**.
- [ ] **Step 4:** Write `spike/FINDINGS.md`: GO/NO-GO verdict; exact selectors for login fields, emulator input, screen text container; observed timings; any 2FA/device-trust behavior; queue command transcript (`QC/`, `Q/<n>`, `*H9*PE*HI`, `*I`, `I`, queue-empty message TEXT verbatim — the engine's stop condition depends on it).
- [ ] **Step 5:** Commit: `git add spike && git commit -m "spike: Sabre Red web automation probe + findings"`

**NO-GO exits:** hard 2FA/CAPTCHA on every login → fallback options per spec §11 (desktop accessibility-tree automation, or Sabre API access via account manager). Stop and report to Mohamed.

---

## Part A — Classy OS module (OS repo, branch `feat/sabre-harvest`)

### Task A1: Schema + migration + grants

**Files:** Create `src/lib/db/schema/sabre-harvest.ts`; modify schema barrel (mirror how `src/lib/db/schema/index.ts` exports existing modules); generate `drizzle/NNNN_sabre_harvest.sql`; modify the permission matrix (`permissionsForRole` source — same file that defines `packages:view` etc.) adding `sabre_harvest:view|edit|run` to `admin` and `manager`.

**Interfaces:** Produces tables per spec §4 exactly: `sabre_credentials`, `sabre_runners`, `sabre_runs`, `sabre_run_contacts` (columns as spec §4; unique index `(run_id, kind, value_normalized)`; all account-scoped with `account_id`). Status enum `queued|running|completed|failed|stopped`.

- [ ] Write the Drizzle schema (mirror column/naming style of an existing schema file, e.g. the hermes tables); statuses as pgEnum; timestamps `created_at/updated_at` per house style.
- [ ] `pnpm drizzle-kit generate` from the ext4 worktree; verify the SQL creates 4 tables + unique index; check number `NNNN` is the next free one.
- [ ] Add the 3 grants to the role matrix; extend the matrix unit test if one asserts grant counts.
- [ ] Run `pnpm test` (full suite must stay green — schema-only change) and commit: `feat(sabre-harvest): schema + grants`.

### Task A2: Credential sealing (crypto)

**Files:** Create `src/lib/sabre-harvest/crypto.ts`, `tests/integration/sabre-harvest-crypto.test.ts`; modify `src/env.ts` (add `SABRE_CRED_KEY` — optional string, validated 32 bytes when base64-decoded).

**Interfaces:** Produces `sealCredentials(plain: {agentId: string; password: string; pcc: string}): string` and `openCredentials(sealed: string): {...}` — versioned format `v1:<iv_b64>:<tag_b64>:<ct_b64>`.

- [ ] **Failing test first:**

```ts
import { sealCredentials, openCredentials } from '@/lib/sabre-harvest/crypto';
it('round-trips and authenticates', () => {
  process.env.SABRE_CRED_KEY = Buffer.alloc(32, 7).toString('base64');
  const sealed = sealCredentials({ agentId: 'A1', password: 'p', pcc: 'AB12' });
  expect(sealed.startsWith('v1:')).toBe(true);
  expect(openCredentials(sealed)).toEqual({ agentId: 'A1', password: 'p', pcc: 'AB12' });
  const tampered = sealed.slice(0, -2) + 'AA';
  expect(() => openCredentials(tampered)).toThrow();
});
```

- [ ] **Implementation:**

```ts
import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';

function key(): Buffer {
  const raw = process.env.SABRE_CRED_KEY;
  if (!raw) throw new Error('SABRE_CRED_KEY not set');
  const k = Buffer.from(raw, 'base64');
  if (k.length !== 32) throw new Error('SABRE_CRED_KEY must be 32 bytes base64');
  return k;
}
export function sealCredentials(plain: { agentId: string; password: string; pcc: string }): string {
  const iv = randomBytes(12);
  const c = createCipheriv('aes-256-gcm', key(), iv);
  const ct = Buffer.concat([c.update(JSON.stringify(plain), 'utf8'), c.final()]);
  return `v1:${iv.toString('base64')}:${c.getAuthTag().toString('base64')}:${ct.toString('base64')}`;
}
export function openCredentials(sealed: string) {
  const [v, ivB, tagB, ctB] = sealed.split(':');
  if (v !== 'v1') throw new Error('unknown credential format');
  const d = createDecipheriv('aes-256-gcm', key(), Buffer.from(ivB, 'base64'));
  d.setAuthTag(Buffer.from(tagB, 'base64'));
  return JSON.parse(Buffer.concat([d.update(Buffer.from(ctB, 'base64')), d.final()]).toString('utf8'));
}
```

- [ ] Run test → PASS; commit `feat(sabre-harvest): AES-256-GCM credential sealing`.

### Task A3: Repository + service layer (pairing, run lifecycle)

**Files:** Create `src/lib/sabre-harvest/repo.ts`, `src/lib/sabre-harvest/service.ts`, `tests/isolation/sabre-harvest.test.ts`.

**Interfaces (produces — later tasks depend on these exact names):**
- `saveCredentials(ctx, {agentId,password,pcc,label})` (edit grant; seals; upsert single row/account; audit)
- `mintPairingCode(ctx): {code, expiresAt}` (edit grant; 8-char code, 10-min TTL, single-use — store hash in `sabre_runners` row with `paired_at` null)
- `pairRunner(code, name): {token}` (UNAUTHED path called by the pair route: burns code, generates 48-byte token, stores sha256 hash, returns plaintext ONCE)
- `authRunner(bearerToken): {runnerId, accountId} | null` (sha256 lookup + `revoked_at is null`; bumps `last_seen_at`)
- `createRun(ctx, {queueNumber,title}): Run` (run grant; **rejects if any run for account in `queued|running`** — spec one-run rule; audit)
- `claimJob(runnerId): {run, credentials} | {stop:true, runId} | null` (first `queued` run for the runner's account → mark `running`, return `openCredentials(...)` payload; also surfaces stop requests for the active run)
- `appendResults(runnerId, runId, rows[]): {inserted}` (validates run ownership+status; `onConflictDoNothing` on the unique key; updates counters + heartbeat)
- `finishRun(runnerId, runId, outcome: 'completed'|'failed'|'stopped', reason?)` → enqueues pg-boss `sabre-harvest.finalize`
- `requestStop(ctx, runId)` (run grant; flags run — delivered via next poll; audit)
- `markStale()` — worker sweep: `running` + heartbeat older than 90s → `failed`, reason `heartbeat timeout`, partial=true, still enqueues finalize.

- [ ] Mirror repo/gating style from an existing module repo (e.g. the operator or finance repo files); every read/mutator calls `requirePermission` except the runner-token paths, which are **service-principal style** — mirror how `src/lib/operator/` builds a scoped context from a hashed bearer (`service_accounts` pattern) and annotate any system reads with the established `// repo-read-ok:` convention where applicable.
- [ ] Isolation tests (mirror `tests/isolation/*`): under-privileged same-account ctx gets `PermissionError` on `saveCredentials`/`createRun`/`requestStop`; second concurrent `createRun` rejects; `authRunner` with wrong/revoked token → null; `appendResults` is idempotent (same batch twice → inserted once).
- [ ] Full suite green; commit `feat(sabre-harvest): repo + service (pairing, run lifecycle)`.

### Task A4: Runner-facing API routes

**Files:** Create the 5 routes under `src/app/api/sabre-runner/**` (paths in File Structure); `tests/integration/sabre-harvest-api.test.ts`.

**Interfaces (JSON contracts — the runner client in Task R2 matches these exactly):**
- `POST /api/sabre-runner/pair` `{code, name}` → `201 {token}` | `401`
- `POST /api/sabre-runner/poll` (Bearer) `{activeRunId?: string}` → `200 {job?: {runId, queueNumber, caps:{maxRecords:5000,maxRuntimeMinutes:240}, credentials:{agentId,password,pcc}}, stop?: boolean}`
- `POST /api/sabre-runner/runs/:id/results` (Bearer) `{rows: [{kind:'phone'|'email', valueNormalized, passengerName?, pnrLocator?, destinationAirport?, destinationCountry?, rawLine, extractedBy:'regex'}], recordsProcessed}` → `200 {inserted}`
- `POST /api/sabre-runner/runs/:id/rescue` (Bearer) `{pnrLocator?, screenText}` → `202`
- `POST /api/sabre-runner/runs/:id/finish` (Bearer) `{outcome:'completed'|'failed'|'stopped', reason?}` → `200`

- [ ] Mirror the `/api/operator` route's auth-first shape (401 before any body parsing side-effects; runtime `nodejs`); zod-validate bodies; add a simple in-memory fixed-window rate limit (60 req/min per token) — the OS has no platform limiter yet.
- [ ] Integration tests: no token → 401 + zero DB writes; happy pair→poll→results→finish flow; results idempotency at the HTTP layer.
- [ ] Suite green; commit `feat(sabre-harvest): runner API`.

### Task A5: Finalization worker — Excel + report

**Files:** Create `src/lib/sabre-harvest/excel.ts`, `src/lib/sabre-harvest/report.ts`; register pg-boss consumer `sabre-harvest.finalize` + the 90s stale sweep in `src/worker/index.ts` (mirror existing sweep registration); `tests/integration/sabre-harvest-excel.test.ts`. Add dep `exceljs`.

**Interfaces:** `buildRunWorkbook(run, contacts): Buffer` — sheets per spec §7: **Contacts** (Type, Value, Passenger Name, PNR, Destination Country, Destination Airport, Queue, Scraped At — Value cells number-format `@`) and **Summary** (counters, status, caps hit, report text). Filename `Sabre Harvest — Queue {n} — YYYY-MM-DD{ — partial}.xlsx`. Upload via the existing R2 documents integration (mirror how M1b docs upload to R2), store key on the run row. `buildReport(run, contacts)` returns deterministic text always; when the LLM adapter is live it may replace it (call through `src/lib/integrations/llm` exactly like an A-series agent does — never construct clients directly), `report_kind` set accordingly.
- [ ] Golden test: fixed run + 3 contacts → workbook parsed back (exceljs read) asserts sheet names, header row, `@` format on Value, and a stable Summary line; report test asserts deterministic text without LLM env.
- [ ] Suite green; commit `feat(sabre-harvest): finalize worker (excel + report)`.

### Task A6: AI rescue handling

**Files:** Modify `src/lib/sabre-harvest/service.ts` (rescue intake → `sabre_run_contacts` row-less holding: store as `needs_review` pseudo-rows with `raw_line = screenText` chunk, `extracted_by='ai'` reserved for parsed output); create worker job `sabre-harvest.rescue` — when LLM live: prompt gpt-4o-mini to emit strict JSON `{contacts:[{kind,value,passengerName?,destinationAirport?}]}` from the screen text, validate with zod, insert via `appendResults` mapping (dedup applies); when mock: leave as `needs_review` for the UI badge. Test with the mock adapter path only (deterministic).
- [ ] Commit `feat(sabre-harvest): AI rescue lane (gated)`.

### Task A7: CRM import (approval-gated)

**Files:** Create `src/lib/sabre-harvest/import.ts`; register a governed action `sabre_contact_import` with executor (mirror an existing action in `src/lib/approvals/` + the governed executor barrel registration — web boot must import it); `tests/integration/sabre-harvest-import.test.ts`.

**Interfaces:** `prepareImport(ctx, runId): {toCreate, toMerge}` (dry summary shown in the approval), executor `executeSabreContactImport(payload)`: for each distinct person in the run — uuidv5 id namespaced `sabre-harvest:<accountId>` over `pnrLocator|valueNormalized`; match existing contacts by normalized phone/email natural keys (mirror `scripts/import/` mappers + `write.ts` conventions): match → merge missing phone/email fields only; no match → create contact `{firstName/lastName from passenger_name, lifecycle:'lead', tags:['import:sabre-harvest-'+runId,'source:sabre', destinationCountry ? 'dest:'+destinationCountry : null]}`; stamp `imported_contact_id` back on run rows.
- [ ] Idempotency test: execute twice → second run inserts 0, merges 0 new values; SoD test: requester cannot approve own import (existing approval-gate test pattern).
- [ ] Suite green; commit `feat(sabre-harvest): approval-gated CRM import`.

### Task A8: Admin UI + i18n

**Files:** Create the two pages + `src/components/sabre-harvest/*` client islands; add dict keys to all 3 locale files (fr canonical) under `sabreHarvest.*`.

- [ ] `/sabre-harvest`: New Run form (queue number remembered via last run, optional title) → server action `createRun`; active run card polling a lightweight server action every 5s (records/phones/emails/last-heartbeat, Stop button → `requestStop`); history table (title, date, counters, status badge, Download → signed R2 URL, Import to CRM button → `prepareImport` + enqueue approval; `needs_review` badge when rescue rows parked). Mirror page pattern: server component → `requireContext('/sabre-harvest')` → section service → permission-gated sections; no function props across the RSC boundary (string-keyed icons).
- [ ] `/sabre-harvest/settings`: credentials form (write-only; shows "credentials set ✓ (updated <date>)" never values) → `saveCredentials`; runner panel (name, last seen, Revoke) + "Generate pairing code" (10-min TTL shown once); download link/instructions for the runner installer.
- [ ] Standard route loading.tsx skeletons; e2e-light smoke per existing pattern if the suite has one for admin pages.
- [ ] Full gates (lint/typecheck/build/vitest/secret-scan) green; commit `feat(sabre-harvest): admin UI + i18n`.

---

## Part B — Runner (this repo)

### Task R1: Runner scaffold + config

**Files:** Create `runner/` per File Structure (`npm init`, deps: `playwright`, dev: `typescript,tsx,vitest,@types/node`); `runner/src/config.ts`.

**Interfaces:** `loadConfig()/saveConfig(c)` — JSON at `%PROGRAMDATA%\SabreHarvest\config.json` (fallback `~/.sabre-harvest/config.json` on dev/Linux): `{osUrl, token, pollSeconds:4}`. Token on disk is acceptable (spec); Sabre creds are NEVER written.
- [ ] Test: round-trip config in a temp dir (env override `SABRE_RUNNER_HOME`). Commit `feat(runner): scaffold + config`.

### Task R2: OS client

**Files:** Create `runner/src/os-client.ts`, `runner/test/os-client.test.ts`.

**Interfaces:** matches Task A4 contracts exactly: `pair(osUrl, code, name)`, `poll(activeRunId?)`, `postResults(runId, rows, recordsProcessed)`, `postRescue(runId, screenText, pnrLocator?)`, `finish(runId, outcome, reason?)`. Retries: exponential backoff ×5 on network/5xx (never on 4xx); `postResults` queues batches in memory and flushes in order so a blip loses nothing.
- [ ] Test against a stub `http.createServer` that fails twice then succeeds → all batches arrive, in order, exactly once. Commit `feat(runner): OS client with durable batching`.

### Task R3: Sabre session driver

**Files:** Create `runner/src/sabre/session.ts`, `runner/src/sabre/mock-server.ts`, `runner/test/session.test.ts`.

**Interfaces:** `openSession({webUrl, agentId, password, pcc, headless}): SabreSession` with `send(command): Promise<string>` (types command + Enter into the emulator input, waits for response settle, returns new screen text), `close()`. Selectors + settle heuristics come from `spike/FINDINGS.md` — wire the real ones; keep them in one `selectors.ts` constant map so a Sabre UI change is a one-file fix.
- [ ] `mock-server.ts`: tiny Express/http server serving a fake login page + emulator textarea that scripts canned responses per command (login → ok; `Q/100` → first PNR; `*H9*PE*HI` → fixture screen; `I` → next/`QUEUE EMPTY` fixture) — this is the CI test double for everything downstream.
- [ ] Test: full login+command round-trip against the mock. Commit `feat(runner): sabre session driver + mock sabre`.

### Task R4: Extraction (port + extend main.py logic)

**Files:** Create `runner/src/extract.ts`, `runner/src/iata-countries.json`, `runner/test/extract.test.ts` with a `fixtures/` dir of captured screens (seed with synthetic ones now; replace with real spike captures before Phase-1 sign-off).

**Interfaces:** `extractRecord(screenText): {rows: ContactRow[], pnrLocator?, passengerName?, destination?: {airport, country}, hadSignals: boolean}` where `ContactRow` matches the A4 results row.

- [ ] Port from `main.py` (same semantics, TS):

```ts
const A9_LINE = /^\s*A9\s+(.*)$/gm;                       // phones live on A9 lines
const PHONE_FLEX = /(?<!\d)(?:1\D*)?(?:\d\D*){10}(?!\d)/g; // 10 digits w/ optional leading 1
const YEN_EMAIL = /¥([^¥]+)¥/g;                            // primary email wrapper
const PLAIN_EMAIL = /\b[A-Za-z0-9._%+\-]+@[A-Za-z0-9.\-]+\.[A-Za-z]{2,}\b/g;
const NAME_LINE = /^\s*1\.1([A-Z' \-]+)\/([A-Z' \-]+?)(?:\s+(?:MR|MRS|MS|MSTR|MISS|DR))?\s*$/m;
// air segment, e.g. " 2 AC 864 Y 12AUG YULCDG HK1 ..." → capture origin+dest IATA pair
const AIR_SEG = /^\s*\d+\s+[A-Z0-9]{2}\s*\d{1,4}[A-Z]?\s+[A-Z]\s+\d{2}[A-Z]{3}\s+([A-Z]{3})([A-Z]{3})/gm;
```

  Phone normalization: strip non-digits; 10 keep; 11 starting `1` → drop the 1; else reject (identical to `main.py`). Email: lowercase, must fullmatch PLAIN_EMAIL. Name: `LAST/FIRST` → `First Last` title-cased.
- [ ] **Destination (spec §6):** collect ordered segment pairs `[orig,dest]`; turnaround = the dest of the last segment whose dest ≠ the itinerary's first origin's country-of-origin chain — implement the simple deterministic rule: walk segments in order; the destination is `dest` of the **last segment before the first segment whose `dest` equals the very first `orig`** (return leg detected); if no such segment, use the final segment's `dest`. Map airport→country via `iata-countries.json` (build once from an open dataset, e.g. OurAirports CSV, checked in — offline at runtime); unknown code → airport set, country null.
- [ ] `hadSignals`: ≥10-digit density or `@` present — feeds the rescue path.
- [ ] Tests: happy phone/email/name/locator; 11-digit phone; junk-rejects; one-way vs round-trip vs open-jaw destination cases; YUL→CDG round trip → France; no itinerary → nulls. Commit `feat(runner): extraction with destination parsing`.

### Task R5: Harvest engine

**Files:** Create `runner/src/engine.ts`, `runner/test/engine.test.ts`.

**Interfaces:** `runHarvest({session, client, job, onLog}): Promise<'completed'|'stopped'|'failed'>`.

- [ ] State machine: `send('Q/'+queueNumber)` → loop per record: `send('*H9*PE*HI')` + `send('*I')` → concat screens → `extractRecord` → rows? `client.postResults(runId, rows, n)` : (hadSignals ? `client.postRescue(...)` : noop) → every loop check caps (`maxRecords`, `maxRuntimeMinutes`) and poll-provided stop → `send('I')` → if screen matches the queue-empty text (verbatim constant from FINDINGS.md, plus generic `QUEUE.*EMPTY`/`NO MORE ITEMS` fallbacks) → `client.finish(runId,'completed')`. Any thrown error → `finish(runId,'failed', message)` after best-effort flush. Heartbeat rides on `poll(activeRunId)` every ~15s from a timer.
- [ ] Test end-to-end vs the mock Sabre (3 records then empty) + a stub OS server: asserts 3 result batches, completed finish, stop-signal honored mid-run, caps trigger `stopped` with reason. Commit `feat(runner): harvest engine`.

### Task R6: CLI + service packaging

**Files:** Create `runner/src/index.ts`, `runner/winsw/sabre-runner.xml`, `runner/README.md`.

- [ ] CLI: `sabre-runner pair <code> --os-url <url> --name <name>` (calls pair, saves config); `sabre-runner run-once` (single poll-claim-harvest, foreground — Phase 1 mode); `sabre-runner serve` (infinite poll loop — service mode). Headless always; `--headed` debug flag.
- [ ] Package: `bun build --compile src/index.ts --outfile dist/sabre-runner.exe` (fallback `@yao-pkg/pkg` if Playwright bundling fights bun; document whichever works). Note: Playwright browser binary must be installed on the PC (`npx playwright install chromium` in install doc) — the exe does not embed Chromium.
- [ ] `winsw/sabre-runner.xml` (service wraps `sabre-runner serve`; logs to `%PROGRAMDATA%\SabreHarvest\logs`); README.md install steps (copy exe + winsw, `sabre-runner pair ...`, `winsw install`, verify Settings page shows Connected).
- [ ] Commit `feat(runner): CLI + Windows service packaging`.

---

## Part C — Integration & rollout

### Task C1: Live integration pass (Phase 1 exit)

- [ ] With Mohamed present: enter real credentials in Settings, pair the runner on the admin PC (`run-once` mode), load a small test queue (2–3 PNRs), Start from the dashboard, watch it complete; verify Excel contents vs the PNRs by eye; capture real screens into `runner/test/fixtures/` (PII-scrubbed synthetics derived from them) and re-run extraction tests.
- [ ] Fix whatever reality broke; only then proceed to Phase 2 tasks (A6/A7 polish, service install).

### Task C2: Reviews + deploy (APPROVAL-GATED)

- [ ] Independent code review + security review over the full OS diff (house pattern: `oh-my-claudecode:code-reviewer` + `security-reviewer`, fix findings, re-run gates).
- [ ] Deploy runbook (execute only with Mohamed's explicit go): prod migration `NNNN` (drizzle-kit migrate via `DATABASE_PUBLIC_URL` from ext4), set `SABRE_CRED_KEY` on web+worker (generate: `openssl rand -base64 32`), deploy web+worker via the ctos-deploy ext4 flow, smoke: `/api/health` 200, `/api/sabre-runner/poll` → 401 unauth, `/sabre-harvest` 307 unauth.
- [ ] Install runner as service on the admin PC; first supervised production harvest; hand over.

## Self-review notes (done at plan time)

- Spec coverage: §3.1→A1/A8, §3.2→R1–R6, §3.3→A5–A7, §4→A1, §5→A4, §6→R4/A6, §7→A5, §8→A3/A4/R5 (stale sweep in A3), §9→A2/A3/A4/C2, §10→tests embedded per task, §11→Part 0/C1/C2, §12 one-run rule→A3 createRun.
- Type consistency: results row shape defined once (A4) and referenced by R2/R4; run statuses single enum; finish outcomes match across A4/R5.
- Known open inputs (not placeholders — external facts): Sabre web URL + EPR (Mohamed), real selectors + queue-empty text (spike outputs, consumed by R3/R5), migration number `NNNN` (checked at build time).
