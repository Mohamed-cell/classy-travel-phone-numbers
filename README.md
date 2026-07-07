# classy-travel-phone-numbers → Sabre Harvest

Extracts customer contact data (phones, emails, names, PNR, final destination)
from Sabre queues for Classy Travel.

## Status (2026-07-06)

- **`main.py`** — the current, working tool: a Windows pyautogui script that
  screen-scrapes the desktop Sabre Red 360 app into `contacts.xlsx`.
  Usage: open Sabre on the queue, display `*H9*PE*HI`, keep the Excel closed,
  `pip install -r requirements.txt`, `python main.py`. Stop: `Ctrl+Alt+K`.
- **Sabre Harvest** — the approved successor: a Classy OS admin feature with an
  invisible headless-browser runner on the admin's PC. **Spec + plan approved,
  implementation NOT started** (deferred on purpose).
  - Spec: [`docs/superpowers/specs/2026-07-06-sabre-harvest-classy-os-design.md`](docs/superpowers/specs/2026-07-06-sabre-harvest-classy-os-design.md)
  - Plan: [`docs/superpowers/plans/2026-07-06-sabre-harvest.md`](docs/superpowers/plans/2026-07-06-sabre-harvest.md)
  - Resume-work prompt: [`docs/superpowers/plans/2026-07-06-sabre-harvest-KICKOFF.md`](docs/superpowers/plans/2026-07-06-sabre-harvest-KICKOFF.md)

To resume the build: open a fresh Claude Code session and paste the prompt from
the KICKOFF file (or just say "continue Sabre Harvest").
