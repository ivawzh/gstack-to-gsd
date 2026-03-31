# Changelog

## 0.2.0 (2026-03-31)

- `/gstack-to-gsd phase N` now auto-triggers `/gsd:plan-phase N` after bridging
- One command goes from gstack reviews to executable GSD plans
- Use `--no-plan` flag to bridge only without triggering planning
- Updated WORKFLOW.md decision tree (merged bridge+plan step)

## 0.1.0 (2026-03-31)

Initial release.

- `/gstack-to-gsd roadmap` — converts gstack CEO plans to GSD ROADMAP.md phases and REQUIREMENTS.md entries
- `/gstack-to-gsd phase N` — converts eng review and design review artifacts to per-phase RESEARCH.md and CONTEXT.md
- `/gstack-to-gsd status` — shows unified state across both systems with "what next?" advice
- `/gstack-to-gsd-upgrade` — update to latest version with auto-check
- `WORKFLOW.md` — paste into CLAUDE.md/AGENTS.md for AI agent routing
