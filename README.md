# gstack-to-gsd

Bridge [gstack](https://github.com/garrytan/gstack) review artifacts into [GSD](https://github.com/vanillacode314/get-shit-done) planning format.

**gstack** designs. **GSD** executes. This bridge connects them.

## What it does

Three commands. Pure artifact translation. No wrapping, no orchestration.

| Command | Input | Output |
|---------|-------|--------|
| `/gstack-to-gsd roadmap` | gstack CEO plan | GSD ROADMAP.md phases + REQUIREMENTS.md |
| `/gstack-to-gsd phase N` | gstack eng + design reviews | Phase RESEARCH.md + CONTEXT.md |
| `/gstack-to-gsd status` | Both systems' state | Terminal advice: what to run next |

## The workflow

```
gstack (strategy)          bridge              GSD (execution)
─────────────────          ──────              ───────────────
/plan-ceo-review    ->  /gstack-to-gsd roadmap  ->  phases in ROADMAP.md
/plan-eng-review    ->  /gstack-to-gsd phase N  ->  RESEARCH.md per phase
/plan-design-review ->  /gstack-to-gsd phase N  ->  CONTEXT.md (UI phases)
                                                     /gsd:plan-phase N
                                                     /gsd:execute-phase N
/qa, /review, /ship                                  (after verification)
```

## Prerequisites

- [gstack](https://github.com/garrytan/gstack) installed (`~/.claude/skills/gstack/`)
- [GSD](https://github.com/vanillacode314/get-shit-done) installed (`~/.claude/get-shit-done/`)
- A GSD project initialized (`.planning/` directory with ROADMAP.md)

## Install (10 seconds)

### Global (all projects)

```bash
git clone --depth 1 https://github.com/ivawzh/gstack-to-gsd.git ~/.claude/skills/gstack-to-gsd-repo
cd ~/.claude/skills/gstack-to-gsd-repo && ./setup
```

### Project-local (committed to repo)

```bash
git clone --depth 1 https://github.com/ivawzh/gstack-to-gsd.git /tmp/gstack-to-gsd
cp -Rf /tmp/gstack-to-gsd/gstack-to-gsd .claude/skills/gstack-to-gsd
rm -rf /tmp/gstack-to-gsd
```

### Add workflow guide to your project

Copy the contents of [WORKFLOW.md](WORKFLOW.md) into your project's `CLAUDE.md` or `AGENTS.md`. This teaches every Claude Code session the full lifecycle and enables the "what should I do next?" routing.

## Data flow

```
~/.gstack/projects/{slug}/
  ceo-plans/*.md                  ─┐
                                   ├─ /gstack-to-gsd roadmap ─> .planning/ROADMAP.md
                                   │                            .planning/REQUIREMENTS.md
                                   │
  *-eng-review-test-plan-*.md     ─┤
  ceo-plans/*.md (arch decisions)  ├─ /gstack-to-gsd phase N ─> .planning/phases/{N}-{name}/{N}-RESEARCH.md
  designs/*/                      ─┘                             .planning/phases/{N}-{name}/{N}-CONTEXT.md
```

## Why this exists

gstack produces excellent strategy documents (CEO/eng/design reviews) but has no execution engine. GSD has a powerful execution pipeline (plan -> execute -> verify) but starts from scratch on strategy. The bridge is the thinnest possible integration: it translates artifacts at the seam, then gets out of the way.

## License

MIT
