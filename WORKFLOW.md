## Development Workflow (gstack + GSD)

This project uses two AI workflow systems together:
- **gstack** — strategy reviews, QA testing, code review, shipping (`/autoplan`, `/plan-ceo-review`, `/plan-eng-review`, `/plan-design-review`, `/qa`, `/review`, `/ship`)
- **GSD** — roadmapping, phase planning, execution, verification (`/gsd:plan-phase`, `/gsd:execute-phase`, `/gsd:verify-work`)
- **`/gstack-to-gsd`** — bridges gstack reviews into GSD and auto-triggers planning

**Lifecycle: Strategy -> Roadmap -> Bridge+Plan -> Build -> Verify -> Ship**

1. STRATEGY (gstack): `/autoplan` or individual reviews (`/plan-ceo-review` + `/plan-eng-review` + `/plan-design-review`)
2. ROADMAP: `/gstack-to-gsd roadmap` (CEO plan -> ROADMAP phases + REQUIREMENTS.md)
3. BRIDGE+PLAN: `/gstack-to-gsd phase N` (bridges eng+design reviews -> RESEARCH.md, then auto-triggers `/gsd:plan-phase N`)
4. BUILD (GSD): `/gsd:execute-phase N`
5. VERIFY (GSD): verification runs inside execute-phase; for manual testing use `/gsd:verify-work N`
6. SHIP (gstack): `/review` then `/ship`

**"What should I do next?" decision tree:**
- No CEO plan in `~/.gstack/projects/`? -> `/autoplan` or `/plan-ceo-review`
- CEO plan exists but phases not in `.planning/ROADMAP.md`? -> `/gstack-to-gsd roadmap`
- Phase in ROADMAP but no `PLAN.md`? -> `/gstack-to-gsd phase N` (bridges + plans in one step)
- Phase has `PLAN.md` but no `SUMMARY.md`? -> `/gsd:execute-phase N`
- Phase verified and all phases done? -> `/review` then `/ship`
- Unsure? -> `/gstack-to-gsd status` (shows unified state)
