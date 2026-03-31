# gstack-to-gsd

Bridge [gstack](https://github.com/garrytan/gstack) review artifacts into [GSD](https://github.com/vanillacode314/get-shit-done) planning format.

**gstack** designs. **GSD** executes. This bridge connects them.

## What it does

Three commands. Translates gstack artifacts into GSD format and auto-triggers planning.

| Command | What it does |
|---------|-------------|
| `/gstack-to-gsd roadmap` | CEO plan -> ROADMAP.md phases + REQUIREMENTS.md |
| `/gstack-to-gsd phase N [M ...]` | Eng+design reviews -> RESEARCH.md, then auto-triggers `/gsd:plan-phase` per phase |
| `/gstack-to-gsd status` | Shows unified state across both systems, recommends next action |

`/gstack-to-gsd phase N` is the main command. One invocation bridges gstack artifacts AND creates GSD execution plans. Accepts multiple phases: `phase 27 28 29` or ranges `phase 27-30`. Use `--no-plan` to bridge only.

## The workflow

```
gstack (strategy)          bridge                    GSD (execution)
─────────────────          ──────                    ───────────────
/plan-ceo-review    ->  /gstack-to-gsd roadmap  ->  phases in ROADMAP.md
/plan-eng-review    ─┐
/plan-design-review  ├─ /gstack-to-gsd phase N  ->  RESEARCH.md + CONTEXT.md
                     │                               then auto-triggers:
                     │                               /gsd:plan-phase N -> PLAN.md
                     │
                     └─────────────────────────────  /gsd:execute-phase N
                                                     /qa, /review, /ship
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

### Update

Run `/gstack-to-gsd-upgrade` in Claude Code, or manually:

```bash
cd ~/.claude/skills/gstack-to-gsd-repo && git pull && ./setup
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
  *-eng-review-test-plan-*.md     ─┤                            .planning/phases/{N}-{name}/
  ceo-plans/*.md (arch decisions)  ├─ /gstack-to-gsd phase N ─>   {N}-RESEARCH.md
  designs/*/                      ─┘                               {N}-CONTEXT.md (UI only)
                                                                   then: /gsd:plan-phase N
                                                                     -> {N}-01-PLAN.md ...
```

## Flags

## Usage examples

```
/gstack-to-gsd phase 27              # bridge + plan one phase
/gstack-to-gsd phase 27 28 29        # bridge + plan three phases sequentially
/gstack-to-gsd phase 27-30           # bridge + plan a range
/gstack-to-gsd phase 27-29 33 35     # mix ranges and individual phases
/gstack-to-gsd phase 28 --no-plan    # bridge only, skip planning
```

| Flag | Command | Effect |
|------|---------|--------|
| `--no-plan` | `phase N` | Bridge only, skip auto-triggering `/gsd:plan-phase` |

## Why this exists

gstack produces excellent strategy documents (CEO/eng/design reviews) but has no execution engine. GSD has a powerful execution pipeline (plan -> execute -> verify) but starts from scratch on strategy. The bridge translates artifacts at the seam and chains the planning step so you get from gstack reviews to executable GSD plans in one command.

## License

MIT
