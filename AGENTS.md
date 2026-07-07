# Sabre Harvest (classy-travel-phone-numbers)

Context file for AI coding agents (Codex/OMX, etc.). Claude Code sessions also
carry this project in persistent memory as `project-sabre-harvest`.

## What this repo is

Sabre queue contact scraper for Classy Travel (Quebec travel agency).
- `main.py` — the CURRENT working tool: Windows pyautogui script that
  screen-scrapes the desktop Sabre Red 360 app into `contacts.xlsx`.
- **Sabre Harvest** — the APPROVED successor (spec'd 2026-07-06, build
  deliberately deferred): a Classy OS admin feature + invisible headless
  Playwright runner on the admin's Windows PC.

## If asked to work on Sabre Harvest

Read these IN ORDER before writing any code:
1. `docs/superpowers/specs/2026-07-06-sabre-harvest-classy-os-design.md` — the approved contract.
2. `docs/superpowers/plans/2026-07-06-sabre-harvest.md` — the task-by-task plan. Its **Global Constraints** section is binding.
3. `docs/superpowers/plans/2026-07-06-sabre-harvest-KICKOFF.md` — resume prompt + human prerequisites.

Start at Task 0.1 (Phase-0 Sabre-web spike, GO/NO-GO). Do not skip it.

## Hard rules

- Phase gates: stop and report at Phase 0 GO/NO-GO, Phase 1 end-to-end, pre-deploy.
- Deploys + prod migrations are APPROVAL-REQUIRED (Mohamed) — never autonomous.
- The Classy OS half of the feature lives in the `classy-travel-ghl-os` repo:
  work it from a fresh ext4 worktree off master (branch `feat/sabre-harvest`),
  NEVER on the `/mnt/c` OS checkout; check the next free drizzle migration
  number at build time (see plan Global Constraints).
- Never commit `*.xlsx` (customer PII — gitignored on purpose).
- No purchases/provisioning of anything billable without explicit approval.
- Human prerequisites before the spike: dedicated Sabre EPR credentials, the
  Sabre Red 360 web login URL, a test queue with 2–3 PNRs.
