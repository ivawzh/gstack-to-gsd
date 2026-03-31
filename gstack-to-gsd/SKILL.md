---
name: gstack-to-gsd
description: "Bridge gstack review artifacts into GSD planning format. Converts CEO plans to ROADMAP phases, eng/design reviews to RESEARCH.md and CONTEXT.md."
user-invocable: true
---

# /gstack-to-gsd — Bridge Gstack Reviews into GSD Plans

Bridge between gstack reviews and GSD execution. Translates artifacts and auto-triggers GSD planning when appropriate.

## Preamble (run first)

```bash
# Update check
_UPD=""
_CHECK=$(find ~/.claude/skills -path "*/gstack-to-gsd*/bin/update-check" -type f 2>/dev/null | head -1)
[ -z "$_CHECK" ] && _CHECK=$(find .claude/skills -path "*/gstack-to-gsd*/bin/update-check" -type f 2>/dev/null | head -1)
[ -n "$_CHECK" ] && _UPD=$(bash "$_CHECK" 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
# Check if updates disabled
_UPDATE_ENABLED=$(cat ~/.gstack-to-gsd/update-check-enabled 2>/dev/null || echo "true")
echo "UPDATE_CHECK: $_UPDATE_ENABLED"
```

If output contains `UPGRADE_AVAILABLE <old> <new>` AND `UPDATE_CHECK` is not `false`:
Read `~/.claude/skills/gstack-to-gsd-repo/gstack-to-gsd-upgrade/SKILL.md` (or find it via the same pattern as above) and follow the "Inline upgrade flow" instructions. Then continue with the command below.

## Data Flow

```
/gstack-to-gsd roadmap:
  IN:  ~/.gstack/projects/{slug}/ceo-plans/*.md        (gstack CEO plan)
  OUT: .planning/ROADMAP.md                             (appended phases)
       .planning/REQUIREMENTS.md                        (appended requirements)

/gstack-to-gsd phase N [M ...] [--no-plan]:
  IN:  ~/.gstack/projects/{slug}/*-eng-review-*.md      (gstack eng review test plan)
       ~/.gstack/projects/{slug}/ceo-plans/*.md          (architecture decisions)
       gstack plan file design review sections           (if /plan-design-review ran)
  OUT: per phase:
       .planning/phases/{N}-{name}/{N}-RESEARCH.md       (architecture + test reqs)
       .planning/phases/{N}-{name}/{N}-CONTEXT.md        (design decisions, UI only)
       then auto-triggers: /gsd:plan-phase N             (unless --no-plan flag)
  Accepts: single (27), multiple (27 28 29), range (27-30), mix (27-29 33)

/gstack-to-gsd status:
  IN:  ~/.gstack/projects/{slug}/*                       (all gstack artifacts)
       .planning/ROADMAP.md, .planning/STATE.md          (GSD state)
  OUT: terminal output only                              (no file writes)
```

## Commands

### `/gstack-to-gsd roadmap`

Converts a gstack CEO plan into GSD ROADMAP.md phases and REQUIREMENTS.md entries.

**Step 1: Find the CEO plan.**

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
SLUG="${SLUG:-unknown}"
CEO_PLAN=$(ls -t ~/.gstack/projects/$SLUG/ceo-plans/*.md 2>/dev/null | head -1)
echo "SLUG: $SLUG"
echo "CEO_PLAN: $CEO_PLAN"
```

If no CEO plan found: stop and say "No gstack CEO plan found. Run `/plan-ceo-review` or `/autoplan` first."

**Step 2: Read the CEO plan.**

Read the full CEO plan file. Extract these sections:
- **Implementation Sequence** — each numbered phase with name, description, acceptance criteria
- **Scope Decisions** — accepted features (the requirements source)
- **Failure Modes & Mitigations** — invert into success criteria negatives
- **Architecture Decisions** — feed into requirements and phase details
- **Milestone Strategy** — where phases fit in the roadmap

**Step 3: Read the existing ROADMAP.md.**

Read `.planning/ROADMAP.md`. Determine:
- The last phase number (e.g., 25)
- The current milestone name and section
- Whether any CEO plan phases already exist (avoid duplicates)

**Step 4: Generate phase entries.**

For each phase in the CEO plan's Implementation Sequence, generate a ROADMAP entry. Map CEO plan structure to GSD format:

```
CEO Plan Field                    → GSD ROADMAP Field
────────────────────────────────  ─────────────────────────
Phase name + number               → ### Phase {N}: {Name}
Phase description (first sentence)→ **Goal**: {description}
"depends on Phase X" references   → **Depends on**: Phase {X}
Acceptance criteria               → **Success Criteria** (what must be TRUE):
Scope decision requirements       → **Requirements**: {IDs}
```

Phase numbering: continue from the last phase in ROADMAP. If CEO plan specifies its own numbering (e.g., "Phase 26"), use that. If CEO plan uses relative numbering (e.g., "Phase 1: Infrastructure"), map to absolute numbers starting from last_phase + 1.

**Success criteria rules:**
- Each acceptance criterion becomes one success criteria line
- Invert relevant failure modes: "Circuit breaker pauses after 3 fails" → "Circuit breaker triggers on 3 consecutive failures and pauses the chore"
- Keep criteria observable and testable (must be verifiable by GSD's verifier agent)

**Dependency rules:**
- Parse "depends on", "requires", "needs", "after" references in CEO plan
- Map CEO plan phase references to absolute phase numbers
- If a dependency references a phase outside the CEO plan (e.g., existing Phase 19), keep the reference as-is

**Step 5: Generate requirement entries.**

For each accepted scope decision in the CEO plan, generate a REQUIREMENTS.md entry:

```markdown
### {ID}: {Name}
**Source**: CEO Plan {date} — Feature {N}
**Phase**: {phase_number}
**Description**: {from scope decision + architecture section details}
```

Requirement ID convention: derive from feature domain.
- Chore features → `CHORE-01`, `CHORE-02`
- QA features → `QA-01`, `QA-02`
- Auth features → `AUTH-01`, `AUTH-02`
- Skill features → `SKILL-01`, `SKILL-02`
- UI features → `UXP-XX` (continue existing series)
- Infrastructure → `INFRA-XX` (continue existing series)

If REQUIREMENTS.md doesn't exist, create it with a header. If it exists, append new entries after existing ones.

**Step 6: Update ROADMAP progress table.**

Append rows to the Progress table for each new phase:
```
| {N}. {Name} | {milestone} | 0/0 | Not started | - |
```

**Step 7: Write files and report.**

Write the updated ROADMAP.md and REQUIREMENTS.md using the Edit tool (append, don't overwrite existing content).

Report:
```
## /gstack-to-gsd roadmap complete

**Source**: {CEO plan filename}
**Phases added**: {N} (Phase {first} through Phase {last})
**Requirements added**: {N} entries

### Phases Created
| Phase | Name | Depends On | Requirements |
|-------|------|------------|--------------|
| 27 | SU-Native Chore Loop | Phase 19 | CHORE-01, CHORE-02 |
| 28 | add-dir Settings | Phase 19 | PROJ-01 |
| ... | ... | ... | ... |

### Next Steps
- /gstack-to-gsd phase 27  — populate RESEARCH.md from eng review
- /gsd:plan-phase 27       — create execution plans
```

---

### `/gstack-to-gsd phase N [M ...] [--no-plan]`

Bridges gstack review artifacts into GSD RESEARCH.md/CONTEXT.md for one or more phases, then auto-triggers `/gsd:plan-phase` for each.

**Accepts multiple phases:**
- Single: `/gstack-to-gsd phase 27`
- Multiple: `/gstack-to-gsd phase 27 28 29`
- Range: `/gstack-to-gsd phase 27-30`
- Mix: `/gstack-to-gsd phase 27-29 33 35`

**Parse ARGUMENTS** to extract all phase numbers. For ranges like `27-30`, expand to `27 28 29 30`. Collect into a list called `PHASES`. Remove `--no-plan` flag if present and set `NO_PLAN=true`.

**For each phase in PHASES, sequentially:**

Run Steps 1-8 below. After each phase completes (bridge + plan), report progress before moving to the next:
```
## Phase {N} of {total} complete
Bridged: {N}-RESEARCH.md
Planned: {plan_count} plan(s)
Remaining: {remaining phases}
```

If any phase fails (missing artifacts, plan-phase error), report the failure and continue to the next phase. Do not stop the entire batch.

**Steps 1-8 below apply per phase.**

**Step 1: Resolve phase.**

```bash
PHASE_DIR=$(ls -d .planning/phases/${N}-* 2>/dev/null | head -1)
echo "PHASE_DIR: $PHASE_DIR"
```

If no phase directory: check ROADMAP.md for the phase. If the phase exists in ROADMAP but has no directory, create it:
```bash
PHASE_NAME=$(grep "Phase ${N}:" .planning/ROADMAP.md | sed 's/.*Phase [0-9]*: //' | sed 's/ *\*\*.*//' | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
mkdir -p ".planning/phases/${N}-${PHASE_NAME}"
```

If phase doesn't exist in ROADMAP either: stop and say "Phase {N} not found. Run `/gstack-to-gsd roadmap` first."

**Step 2: Find gstack artifacts.**

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
SLUG="${SLUG:-unknown}"

# Eng review test plan (most recent)
ENG_PLAN=$(ls -t ~/.gstack/projects/$SLUG/*-eng-review-test-plan-*.md 2>/dev/null | head -1)

# CEO plan (for architecture decisions)
CEO_PLAN=$(ls -t ~/.gstack/projects/$SLUG/ceo-plans/*.md 2>/dev/null | head -1)

# Design review artifacts
DESIGN_DIR=$(ls -td ~/.gstack/projects/$SLUG/designs/*/ 2>/dev/null | head -1)

echo "ENG_PLAN: ${ENG_PLAN:-none}"
echo "CEO_PLAN: ${CEO_PLAN:-none}"
echo "DESIGN_DIR: ${DESIGN_DIR:-none}"
```

If no eng review AND no CEO plan: stop and say "No gstack review artifacts found. Run `/plan-eng-review` first, then `/gstack-to-gsd phase {N}`."

**Step 3: Extract phase-relevant content from CEO plan.**

Read the CEO plan. Find sections relevant to phase N:
- Architecture decisions mentioning this phase's domain
- Implementation sequence entry for this phase (acceptance criteria, approach)
- Failure modes relevant to this phase
- Research findings applied to this phase's domain

**Step 4: Extract eng review findings.**

If eng review test plan exists, read it and extract:
- Affected pages/routes relevant to this phase
- Key interactions to verify
- Edge cases identified
- Critical paths
- Architecture findings (dependency graph, coupling concerns)

**Step 5: Write RESEARCH.md.**

Write to `{phase_dir}/{N}-RESEARCH.md`:

```markdown
# Phase {N}: {Name} - Research

**Source**: gstack eng review ({date}) + CEO plan architecture decisions
**Bridged**: {today's date}

## Summary
{Phase goal from ROADMAP + context from CEO plan implementation sequence}

## Architecture (from eng review)
{Architecture findings relevant to this phase}
{Dependency graph excerpts}

## Test Requirements (from eng review)
{Test plan items scoped to this phase}
- {Affected routes}
- {Key interactions}
- {Edge cases}

## Key Decisions (from CEO plan)
{Architecture decisions section, filtered to this phase's domain}
- {Decision 1}: {rationale}
- {Decision 2}: {rationale}

## Failure Modes (from CEO plan)
{Failure modes table entries relevant to this phase}

## Patterns to Follow
{Research findings from CEO plan relevant to this phase's domain}
```

**Step 6: Write CONTEXT.md (UI phases only).**

Detect UI phase: check if the phase's ROADMAP entry mentions UI, frontend, page, dashboard, gallery, view, or component.

If UI phase AND design review artifacts exist:

Write to `{phase_dir}/{N}-CONTEXT.md`:

```markdown
# Phase {N}: {Name} - Context

**Source**: gstack design review + CEO plan
**Bridged**: {today's date}

## Phase Boundary
{From ROADMAP: goal + success criteria}

## Design Decisions (from gstack design review)
{Design specs, scores per dimension, state coverage requirements}
{Mockup references if they exist}

## Implementation Decisions (from CEO plan)
{Locked decisions relevant to this phase}
- {Decision}: {rationale}

## Existing Code Insights
{Note: run /gsd:discuss-phase {N} to populate with codebase analysis}
```

If NOT a UI phase: skip CONTEXT.md (GSD's discuss-phase will create it from codebase analysis).

**Step 7: Report bridge results.**

```
## /gstack-to-gsd phase {N} — bridge complete

**Phase**: {N} - {Name}
**Written**:
  - {phase_dir}/{N}-RESEARCH.md (from eng review + CEO plan)
  - {phase_dir}/{N}-CONTEXT.md (from design review) [if UI phase]

Launching /gsd:plan-phase {N}...
```

**Step 8: Auto-trigger GSD planning.**

**If `--no-plan` flag is present in ARGUMENTS:** Skip. Report "Bridge complete. Run `/gsd:plan-phase {N}` when ready." and stop.

**Otherwise:** Invoke GSD plan-phase immediately using the Skill tool:

```
Skill(skill="gsd:plan-phase", args="{N}")
```

This creates the full bridge-to-plan pipeline in one command: gstack artifacts are translated to RESEARCH.md/CONTEXT.md, then GSD's planner reads them and produces executable PLAN.md files.

---

### `/gstack-to-gsd status`

Shows unified state across both systems. No file writes.

**Step 1: Discover gstack artifacts.**

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
SLUG="${SLUG:-unknown}"
echo "=== GSTACK ARTIFACTS ==="
echo "CEO plans:"
ls -t ~/.gstack/projects/$SLUG/ceo-plans/*.md 2>/dev/null || echo "  none"
echo "Eng reviews:"
ls -t ~/.gstack/projects/$SLUG/*-eng-review-*.md 2>/dev/null || echo "  none"
echo "Design artifacts:"
ls -td ~/.gstack/projects/$SLUG/designs/*/ 2>/dev/null || echo "  none"
```

**Step 2: Read GSD state.**

Read `.planning/ROADMAP.md` and `.planning/STATE.md`. Extract:
- Total phases, last phase number
- Which phases have RESEARCH.md, CONTEXT.md, PLAN.md, SUMMARY.md
- Current position from STATE.md

**Step 3: Cross-reference.**

For each phase in ROADMAP that came from a CEO plan:
- Check if RESEARCH.md exists (bridged from eng review?)
- Check if CONTEXT.md exists (bridged from design review?)
- Check if PLAN.md exists (GSD planned?)
- Check if SUMMARY.md exists (GSD executed?)

**Step 4: Display status.**

```
## Gstack-to-GSD Status

### Gstack Artifacts
  CEO Plan: {filename} ({date}, {N} features)
  Eng Review: {filename or "not run"}
  Design Review: {status}

### Bridged Phases
| Phase | Name | RESEARCH | CONTEXT | PLAN | EXECUTED |
|-------|------|----------|---------|------|----------|
| 26 | Event Bus Standardization | - | - | Yes | Yes |
| 27 | SU-Native Chore Loop | No | No | No | No |
| 28 | add-dir Settings | No | No | No | No |
| ... | ... | ... | ... | ... | ... |

### What to Do Next
{Based on the state, recommend the single next action:}

- If phases not in ROADMAP: "/gstack-to-gsd roadmap — add CEO plan phases"
- If phase has no RESEARCH.md and no PLAN.md: "/gstack-to-gsd phase N — bridge + plan in one step"
- If phase has PLAN.md but no SUMMARY.md: "/gsd:execute-phase N — execute plans"
- If phase verified and all phases done: "/review then /ship"
- If no eng review exists: "/plan-eng-review — run eng review first"
```

---

## Error Handling

- **No gstack slug**: Fall back to repo basename. Warn: "Could not detect gstack project slug."
- **CEO plan parse failure**: If Implementation Sequence section not found, try parsing numbered lists or phase headers. If still fails: "CEO plan format not recognized. Expected '## Implementation Sequence' section."
- **Phase already exists in ROADMAP**: Skip it, report "Phase {N} already exists, skipping."
- **RESEARCH.md already exists**: Ask "RESEARCH.md exists for phase {N}. Overwrite? (y/n)" — default: no.
- **No eng review for phase**: Write RESEARCH.md from CEO plan only (architecture decisions + failure modes). Note: "Partial bridge — CEO plan only. Run `/plan-eng-review` for full coverage."
