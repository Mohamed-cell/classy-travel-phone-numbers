# Sabre Harvest — Kickoff Prompt (paste into a fresh Claude Code session)

> Status when written (2026-07-06): spec + plan APPROVED by Mohamed; ZERO implementation done.
> Deferred deliberately (weekly usage budget). Everything needed is in this repo + Claude's memory.

## The prompt

```
Continue the Sabre Harvest project.

1. Read /mnt/c/projects/classy-travel-phone-numbers/docs/superpowers/specs/2026-07-06-sabre-harvest-classy-os-design.md (the approved spec — the contract).
2. Read /mnt/c/projects/classy-travel-phone-numbers/docs/superpowers/plans/2026-07-06-sabre-harvest.md (the implementation plan — the route). Honor its Global Constraints section completely.
3. Execute the plan with superpowers:subagent-driven-development, task by task, starting with Task 0.1 (the Phase-0 Sabre-web spike, GO/NO-GO). Do NOT skip the spike.
4. Before the spike, confirm with me: the dedicated Sabre EPR credentials, the Sabre Red 360 web login URL, and the test queue number.
5. Stop and report at every phase gate (Phase 0 GO/NO-GO, Phase 1 end-to-end, pre-deploy). Deploys and prod migrations remain approval-gated — never autonomous.
```

## Human prerequisites (Mohamed, before/while the spike runs)

- [ ] Create the dedicated Sabre agent sign-in (EPR) for the bot
- [ ] Provide the Sabre Red 360 **web** login URL the agency uses
- [ ] Pick the test queue and put 2–3 PNRs on it

## Where things stand

- Repo `classy-travel-phone-numbers`: legacy pyautogui scraper (`main.py`, still functional as manual fallback), approved spec + plan in `docs/superpowers/`.
- Classy OS side: nothing built; will be branch `feat/sabre-harvest` in classy-travel-ghl-os, worked from a fresh ext4 worktree (see plan Global Constraints — migration numbering + DrvFs gotchas).
- Claude memory key: `project-sabre-harvest` (auto-recalled by mentioning "Sabre Harvest").
