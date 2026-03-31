---
name: gstack-to-gsd-upgrade
description: "Upgrade gstack-to-gsd to the latest version. Detects install type, pulls updates, re-runs setup, shows what's new."
user-invocable: true
---

# /gstack-to-gsd-upgrade

Upgrade gstack-to-gsd to the latest version and show what's new.

## Inline upgrade flow

Referenced by the main skill's preamble when `UPGRADE_AVAILABLE` is detected.

### Step 1: Ask the user

Use AskUserQuestion:
- Question: "gstack-to-gsd **v{new}** is available (you're on v{old}). Upgrade now?"
- Options: ["Yes, upgrade now", "Not now", "Never ask again"]

**If "Yes":** Proceed to Step 2.

**If "Not now":** Write snooze state (escalating: 24h, 48h, 1 week):
```bash
SNOOZE_FILE=~/.gstack-to-gsd/update-snoozed
REMOTE_VER="{new}"
CUR_LEVEL=0
if [ -f "$SNOOZE_FILE" ]; then
  SNOOZED_VER=$(awk '{print $1}' "$SNOOZE_FILE")
  if [ "$SNOOZED_VER" = "$REMOTE_VER" ]; then
    CUR_LEVEL=$(awk '{print $2}' "$SNOOZE_FILE")
    case "$CUR_LEVEL" in *[!0-9]*) CUR_LEVEL=0 ;; esac
  fi
fi
NEW_LEVEL=$((CUR_LEVEL + 1))
[ "$NEW_LEVEL" -gt 3 ] && NEW_LEVEL=3
echo "$REMOTE_VER $NEW_LEVEL $(date +%s)" > "$SNOOZE_FILE"
```
Continue with original skill.

**If "Never ask again":**
```bash
mkdir -p ~/.gstack-to-gsd
echo "false" > ~/.gstack-to-gsd/update-check-enabled
```
Tell user: "Update checks disabled. Delete `~/.gstack-to-gsd/update-check-enabled` to re-enable."

### Step 2: Detect install type

```bash
if [ -d "$HOME/.claude/skills/gstack-to-gsd-repo/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.claude/skills/gstack-to-gsd-repo"
elif [ -d ".claude/skills/gstack-to-gsd-repo/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".claude/skills/gstack-to-gsd-repo"
elif [ -d ".claude/skills/gstack-to-gsd" ]; then
  INSTALL_TYPE="vendored"
  INSTALL_DIR=".claude/skills/gstack-to-gsd"
elif [ -d "$HOME/.claude/skills/gstack-to-gsd" ]; then
  INSTALL_TYPE="vendored-global"
  INSTALL_DIR="$HOME/.claude/skills/gstack-to-gsd"
else
  echo "ERROR: gstack-to-gsd not found"
  exit 1
fi
echo "Install type: $INSTALL_TYPE at $INSTALL_DIR"
```

### Step 3: Save old version

```bash
OLD_VERSION=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
```

### Step 4: Upgrade

**For git installs** (global-git, local-git):
```bash
cd "$INSTALL_DIR"
git fetch origin
git reset --hard origin/main
./setup
```

**For vendored installs** (vendored, vendored-global):
```bash
TMP_DIR=$(mktemp -d)
git clone --depth 1 https://github.com/ivawzh/gstack-to-gsd.git "$TMP_DIR/gstack-to-gsd"
# Copy only the skill directory content (vendored installs don't have the full repo)
cp -f "$TMP_DIR/gstack-to-gsd/gstack-to-gsd/SKILL.md" "$INSTALL_DIR/SKILL.md"
cp -f "$TMP_DIR/gstack-to-gsd/VERSION" "$INSTALL_DIR/VERSION" 2>/dev/null || true
rm -rf "$TMP_DIR"
```

### Step 5: Clear cache

```bash
mkdir -p ~/.gstack-to-gsd
rm -f ~/.gstack-to-gsd/last-update-check
rm -f ~/.gstack-to-gsd/update-snoozed
```

### Step 6: Show what's new

Read the new VERSION:
```bash
NEW_VERSION=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
```

Tell user:
```
gstack-to-gsd v{NEW_VERSION} — upgraded from v{OLD_VERSION}!

Run `/gstack-to-gsd status` to see current state.
```

## Standalone usage

When invoked directly as `/gstack-to-gsd-upgrade`:

1. Force update check:
```bash
SCRIPT_DIR=$(find ~/.claude/skills -path "*/gstack-to-gsd*/bin/update-check" -type f 2>/dev/null | head -1)
[ -z "$SCRIPT_DIR" ] && SCRIPT_DIR=$(find .claude/skills -path "*/gstack-to-gsd*/bin/update-check" -type f 2>/dev/null | head -1)
[ -n "$SCRIPT_DIR" ] && bash "$SCRIPT_DIR" --force
```

2. If `UPGRADE_AVAILABLE`: follow Steps 2-6 above.
3. If no output: "You're on the latest version."
