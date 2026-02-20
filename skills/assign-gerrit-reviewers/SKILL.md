# assign-gerrit-reviewers

Automatically assigns Gerrit reviewers to your open changes every hour based on CODEOWNERS and a user-supplied team roster.

## What it does

On a schedule (every hour), this skill:

1. Finds your open Gerrit changes that are currently `Verified+1` (passing CI)
2. Skips any change that already has at least one human reviewer assigned
3. Reads the repo's CODEOWNERS file to determine which team owns the most changed files
4. Looks up that team's members from your roster file
5. Assigns all team members as reviewers via `ger`

## Setup

### 1. Create a team roster file

Create a file at `~/.config/assign-gerrit-reviewers/teams.csv` (or YAML/JSON — see formats below) that maps team names to member Gerrit emails.

**CSV format** (`teams.csv`):
```csv
team,email
frontend,alice@example.com
frontend,bob@example.com
backend,carol@example.com
backend,dave@example.com
```

**YAML format** (`teams.yaml`):
```yaml
frontend:
  - alice@example.com
  - bob@example.com
backend:
  - carol@example.com
  - dave@example.com
```

**Markdown table format** (`teams.md`):
```markdown
| team     | email               |
|----------|---------------------|
| frontend | alice@example.com   |
| frontend | bob@example.com     |
| backend  | carol@example.com   |
```

The team names here must match (case-insensitively) what appears in your CODEOWNERS file.

### 2. Configure the skill

Create `~/.config/assign-gerrit-reviewers/config.yaml`:

```yaml
# Path to your team roster file (CSV, YAML, or Markdown)
roster: ~/.config/assign-gerrit-reviewers/teams.csv

# What to do when no CODEOWNERS match is found for a change.
# Options:
#   skip         - do nothing, leave the change unassigned
#   assign_team  - assign a specific fallback team (set fallback_team below)
no_match_behavior: assign_team
fallback_team: frontend   # only used if no_match_behavior is assign_team

# Bot detection: accounts matching any of these patterns are considered bots
# and are NOT counted as human reviewers.
# Matching is case-insensitive, substring match on display name or email.
bot_patterns:
  - Jenkins
  - "(Bot)"
  - "Larry Gergich"
  - "bot@"
  - "noreply"
```

### 3. Install the cron job

Run this once to register the hourly job:

```bash
# Using cron
(crontab -l 2>/dev/null; echo "0 * * * * /path/to/assign-gerrit-reviewers.sh >> ~/.config/assign-gerrit-reviewers/run.log 2>&1") | crontab -

# Or using launchd on macOS — create ~/Library/LaunchAgents/com.user.assign-gerrit-reviewers.plist
```

## How CODEOWNERS matching works

For each changed file in the change, the skill finds the most specific matching CODEOWNERS rule and records which team owns it. The team with the most owned files wins. If there's a tie, the first team encountered wins.

CODEOWNERS team entries like `@org/team-name` are matched against your roster using the part after `/` (e.g. `team-name`).

## How bot detection works

An account is considered a bot if its display name or email **contains** any of the configured `bot_patterns` (case-insensitive). Defaults catch common patterns like `Jenkins`, `(Bot)`, and `Larry Gergich`. You can extend the list in your config.

## Running manually

```bash
assign-gerrit-reviewers.sh
```

Or with dry-run to preview what would be assigned without making changes:

```bash
assign-gerrit-reviewers.sh --dry-run
```

## The script

Save as `~/bin/assign-gerrit-reviewers.sh` (or wherever you prefer) and make it executable:

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/assign-gerrit-reviewers"
CONFIG="$CONFIG_DIR/config.yaml"
DRY_RUN=false

[[ "${1:-}" == "--dry-run" ]] && DRY_RUN=true

# --- helpers ---

log() { echo "[assign-gerrit-reviewers] $*"; }

is_bot() {
  local name="$1"
  # Read bot_patterns from config; fall back to built-in defaults
  local patterns
  patterns=$(yq -r '.bot_patterns[]? // empty' "$CONFIG" 2>/dev/null) || true
  if [[ -z "$patterns" ]]; then
    patterns=$'Jenkins\n(Bot)\nLarry Gergich\nbot@\nnoreply'
  fi
  while IFS= read -r pat; do
    [[ -z "$pat" ]] && continue
    if echo "$name" | grep -qi "$pat"; then
      return 0
    fi
  done <<< "$patterns"
  return 1
}

has_human_reviewer() {
  local change_id="$1"
  # ger reviewers returns one reviewer per line as "Display Name <email>"
  local reviewers
  reviewers=$(ger reviewers "$change_id" 2>/dev/null) || return 1
  while IFS= read -r reviewer; do
    [[ -z "$reviewer" ]] && continue
    if ! is_bot "$reviewer"; then
      return 0  # found a human
    fi
  done <<< "$reviewers"
  return 1
}

get_team_for_change() {
  local change_id="$1"
  # ger files returns changed file paths, one per line
  local files
  files=$(ger files "$change_id" 2>/dev/null) || { echo ""; return; }

  declare -A team_counts
  local codeowners
  codeowners=$(ger codeowners 2>/dev/null || cat CODEOWNERS 2>/dev/null || true)

  while IFS= read -r file; do
    [[ -z "$file" ]] && continue
    # Find the last matching CODEOWNERS rule for this file
    local owner=""
    while IFS= read -r rule; do
      [[ -z "$rule" || "$rule" == \#* ]] && continue
      local pattern team_entry
      pattern=$(echo "$rule" | awk '{print $1}')
      team_entry=$(echo "$rule" | awk '{print $2}')
      # Simple glob match
      if [[ "$file" == $pattern* ]] || [[ "$file" == */$pattern ]] || [[ "$file" == $pattern ]]; then
        owner="$team_entry"
      fi
    done <<< "$codeowners"
    if [[ -n "$owner" ]]; then
      # Strip @org/ prefix
      local team_name="${owner##*/}"
      team_counts["$team_name"]=$(( ${team_counts["$team_name"]:-0} + 1 ))
    fi
  done <<< "$files"

  # Return the team with the most files
  local best_team="" best_count=0
  for t in "${!team_counts[@]}"; do
    if (( team_counts[$t] > best_count )); then
      best_count=${team_counts[$t]}
      best_team="$t"
    fi
  done
  echo "$best_team"
}

get_team_members() {
  local team="$1"
  local roster
  roster=$(yq -r '.roster' "$CONFIG" 2>/dev/null | sed "s|~|$HOME|g") || return 1

  local ext="${roster##*.}"
  case "$ext" in
    csv)
      # team,email CSV
      awk -F',' -v t="$team" 'NR>1 && tolower($1)==tolower(t) {print $2}' "$roster"
      ;;
    yaml|yml)
      yq -r ".[\"$team\"][]?" "$roster" 2>/dev/null
      ;;
    md|markdown)
      # Markdown table: | team | email |
      awk -F'|' -v t="$team" '
        NF>=3 && tolower($2) ~ tolower(t) {
          gsub(/^[ \t]+|[ \t]+$/, "", $3); print $3
        }
      ' "$roster"
      ;;
  esac
}

# --- main ---

log "Checking for changes to assign..."

# Get my open changes that are Verified+1 and have no Code-Review vote yet
mapfile -t changes < <(ger mine --status open --label "Verified+1" --format id 2>/dev/null)

if [[ ${#changes[@]} -eq 0 ]]; then
  log "No eligible changes found."
  exit 0
fi

no_match_behavior=$(yq -r '.no_match_behavior // "skip"' "$CONFIG" 2>/dev/null || echo "skip")
fallback_team=$(yq -r '.fallback_team // ""' "$CONFIG" 2>/dev/null || echo "")

for change_id in "${changes[@]}"; do
  [[ -z "$change_id" ]] && continue

  if has_human_reviewer "$change_id"; then
    log "[$change_id] Already has human reviewer, skipping."
    continue
  fi

  team=$(get_team_for_change "$change_id")

  if [[ -z "$team" ]]; then
    case "$no_match_behavior" in
      skip)
        log "[$change_id] No CODEOWNERS match, skipping."
        continue
        ;;
      assign_team)
        if [[ -z "$fallback_team" ]]; then
          log "[$change_id] No CODEOWNERS match and no fallback_team configured, skipping."
          continue
        fi
        team="$fallback_team"
        log "[$change_id] No CODEOWNERS match, falling back to team: $team"
        ;;
    esac
  fi

  mapfile -t members < <(get_team_members "$team")

  if [[ ${#members[@]} -eq 0 ]]; then
    log "[$change_id] Team '$team' has no members in roster, skipping."
    continue
  fi

  log "[$change_id] Assigning team '$team' (${#members[@]} members)..."

  for email in "${members[@]}"; do
    [[ -z "$email" ]] && continue
    if $DRY_RUN; then
      log "  [dry-run] would assign: $email"
    else
      ger assign-reviewer "$change_id" "$email" 2>/dev/null \
        && log "  Assigned: $email" \
        || log "  Failed to assign: $email"
    fi
  done
done

log "Done."
```

## Dependencies

- `ger` CLI — for querying Gerrit changes, files, reviewers, and assigning (https://github.com/aaronshaf/ger)
- `yq` — for parsing YAML config and roster (`brew install yq` or `apt install yq`)
- `awk`, `bash` — standard shell tools

## Notes on `ger` commands used

This skill assumes the following `ger` subcommands exist (adjust if your version differs):

| Command | Purpose |
|---|---|
| `ger mine --status open --label "Verified+1" --format id` | List your open Verified+1 change IDs |
| `ger reviewers <change-id>` | List reviewers as "Name \<email\>" |
| `ger files <change-id>` | List changed file paths |
| `ger codeowners` | Print repo CODEOWNERS (or fall back to local `CODEOWNERS` file) |
| `ger assign-reviewer <change-id> <email>` | Add a reviewer |

Check `ger help` and adjust the command flags to match your installed version.
