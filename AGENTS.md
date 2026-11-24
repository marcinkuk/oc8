# Agent instructions

Guidance for AI coding agents working in this repository.

- Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a pull request —
  it covers the DCO sign-off, coding guidelines, and PR process.
- Before committing: `cd backend && python -m pytest && python -m ruff check src tests`
  and `cd frontend && npm run lint && npm run test:unit && npm run build`.
- Follow existing patterns in the module you're touching rather than
  introducing a new one — see [docs/contributing/coding-guidelines.md](docs/contributing/coding-guidelines.md).
- Telegram approvals: [docs/TELEGRAM_APPROVALS.md](docs/TELEGRAM_APPROVALS.md)
  covers where the bot token lives, the outbound-transport design (address
  ranking + connect-only retry — do not simplify it away), deploying the live
  Portainer stack, and how to health-check polling.
- Known failures that exist on plain upstream too (don't chase them): backend
  `tests/approvals/test_reads_go_through_the_scoped_repository.py::test_no_module_loads_an_approval_or_a_clarification_outside_the_repo`
  (`agents/retire.py` loads `ApprovalRequest`); frontend `tsc --noEmit` reports
  8 errors (mission-automation, capa-export-wizard); `totp-challenge`
  resend-timer test fails intermittently under full-suite load.
- Upstream is `oc8-ai/oc8`; contributions need Conventional Commits plus a
  `Signed-off-by` DCO line (`git commit -s`).
