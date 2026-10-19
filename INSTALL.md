# Install: project memory for Claude Code

This file is a complete, self-contained installer for Claude Code. Give it to Claude Code and say:
**"Install this memory system."** Nothing else is needed: every file is embedded below.

What gets installed (see `README.md` for the concept):

- Scripts in `~/.claude/memory-system/`, the same on every machine.
- Hook entries and one permission rule merged into `~/.claude/settings.json`.
- A rules block in `~/.claude/CLAUDE.md`, between markers.
- Optional: the `/memory-sync` skill in `~/.claude/skills/memory-sync/`.
- Per project: a `memory/` folder, created only when you enable that project.

---

## Instructions for Claude (read before doing anything)

1. Follow the steps **in order**. Report the result of each step in one or two lines.
2. Write every embedded file **byte-exact**: copy everything between the `~~~~` fence lines exactly. Do not reformat, fix, translate, shorten, or "improve" anything.
   - Keep backslashes, quotes, tabs and blank lines as they are.
   - The `~~~~` fence lines themselves are not part of the file.
3. Use the Write tool (create parent folders as needed). If you must use a shell, use a quoted heredoc (`cat > file <<'EOF'`) so nothing is expanded.
4. Do not edit `~/.claude/settings.json` or `~/.claude/CLAUDE.md` by hand. `install.sh` (Step 3) merges into them safely and writes backups first.
5. Ask the user **only** the questions in Step 1. Use the defaults when they say so.
6. If a command fails, stop. Show the error and the step it happened in. Do not improvise workarounds that change the design.
7. Never write secrets, tokens or passwords into any file.
8. `~` means the user's home directory. On Windows (Git Bash) that is `%USERPROFILE%`, which Git Bash shows as `/c/Users/<name>`.

---

## Step 0: Check the environment

Detect the OS:

```bash
uname -s; [ -r /proc/version ] && grep -qi microsoft /proc/version && echo WSL
```

| `uname -s` | Environment |
|---|---|
| `Darwin` | macOS |
| `Linux` (+ `WSL`) | Linux, or Windows through WSL. Install everything inside WSL, where Claude Code runs. |
| `MINGW*` / `MSYS*` / `CYGWIN*` | Windows with Git Bash (native Claude Code on Windows uses Git Bash) |

Check the required tools:

```bash
for c in bash git jq awk sed find tar mktemp; do command -v "$c" >/dev/null 2>&1 && echo "ok: $c" || echo "MISSING: $c"; done
command -v rsync >/dev/null 2>&1 && echo "ok: rsync" || echo "optional: rsync not found (sync falls back to tar)"
```

If anything is `MISSING`, stop and give the user the install command for their OS. Continue after they confirm it is installed.

| OS | Command |
|---|---|
| macOS | `brew install jq` (git: `xcode-select --install`; bash, awk, sed, tar are built in) |
| Debian/Ubuntu/WSL | `sudo apt-get install -y jq git rsync` |
| Fedora/RHEL | `sudo dnf install -y jq git rsync` |
| Arch | `sudo pacman -S jq git rsync` |
| Windows (Git Bash) | Install Git for Windows (gives bash, awk, sed, tar). Then `winget install jqlang.jq` and restart the terminal and Claude Code so `jq` is on PATH. rsync is not needed. |

Windows note: the hooks run as `bash ~/.claude/memory-system/hooks/...`, so `bash` (Git Bash) must be on PATH. The default Git for Windows install handles this. If hooks later fail with "bash: command not found", add `C:\Program Files\Git\bin` to the PATH.

**Existing install?** If `~/.claude/memory-system/` already exists, this run is an **update**. The files are overwritten and `install.sh` re-merges safely. Projects' `memory/` folders are never touched.

---

## Step 1: Ask the user

Ask these questions together, in one message:

1. **Git author email**, for the rule "only record my own work".
   - Suggest the output of `git config --global user.email`.
   - If the user doesn't use git, use their name or any label.
2. **Install multi-machine sync (`/memory-sync`)?** Default: yes.
3. If yes: **sync folder**. Default: `~/claude-memory-sync`.
   - This is a local git working copy that the user later connects to a private remote.
   - The folder can differ per machine.

---

## Step 2: Write the files

Write each file below to the path shown above its block (19 files). Create folders as needed.

| # | Path | Purpose |
|---|---|---|
| 1 | `~/.claude/memory-system/hooks/lib.sh` | Shared helpers (paths, state folder, week names, ID to file) |
| 2 | `~/.claude/memory-system/hooks/log-prompt.sh` | UserPromptSubmit: logs each prompt as a history entry; asks for last week's summary |
| 3 | `~/.claude/memory-system/hooks/log-edit.sh` | PostToolUse (Write/Edit): records files Claude edited this turn |
| 4 | `~/.claude/memory-system/hooks/check-memory.sh` | Stop: blocks the turn if a note or Did is missing; rebuilds lookups; commits memory/ |
| 5 | `~/.claude/memory-system/hooks/build-refs.sh` | Generates memory/REFS.md and memory/refs/YYYY-MM.md |
| 6 | `~/.claude/memory-system/hooks/build-global-index.sh` | Generates ~/.claude/memory-global/GLOBAL_INDEX.md (notes across projects) |
| 7 | `~/.claude/memory-system/hooks/session-status.sh` | SessionStart: shows whether memory is enabled |
| 8 | `~/.claude/memory-system/hooks/guard-memory.sh` | PreToolUse (Write/Edit): blocks writes into another project's memory/ |
| 9 | `~/.claude/memory-system/hooks/guard-memory-read.sh` | PreToolUse/PostToolUse (Read/Grep/Glob/Bash): asks before reading another project's memory; runs the history gate |
| 10 | `~/.claude/memory-system/hooks/guard-history.sh` | History gate: grep/read of history only after the lookup script ran this turn |
| 11 | `~/.claude/memory-system/lookup-memory.sh` | The lookup command (notes + every history entry) |
| 12 | `~/.claude/memory-system/init-memory.sh` | Enable / disable / remove memory for a project |
| 13 | `~/.claude/memory-system/sync-memory.sh` | Multi-machine sync through a git repo |
| 14 | `~/.claude/memory-system/install.sh` | Merges settings.json, writes the CLAUDE.md rules block, installs the skill; also uninstalls |
| 15 | `~/.claude/memory-system/selftest.sh` | Sandbox self-test |
| 16 | `~/.claude/memory-system/templates/INDEX.md` | Template: a project's memory/INDEX.md |
| 17 | `~/.claude/memory-system/templates/TAGS.md` | Template: a project's memory/TAGS.md |
| 18 | `~/.claude/memory-system/templates/CLAUDE-section.md` | The rules block for ~/.claude/CLAUDE.md ({{GIT_EMAIL}} is filled in by install.sh) |
| 19 | `~/.claude/memory-system/templates/memory-sync-SKILL.md` | The /memory-sync skill (copied by install.sh) |

### File 1: `~/.claude/memory-system/hooks/lib.sh`

~~~~bash
# Shared helpers, sourced by the memory hooks. Not executed directly.
# Path to POSIX form. On Windows (Git Bash / MSYS) hook JSON and CLAUDE_PROJECT_DIR carry
# C:\... paths; cygpath turns them into /c/... so they compare with $HOME and each other.
# Elsewhere cygpath does not exist and the path passes through unchanged.
norm() {
  if [ -n "$1" ] && command -v cygpath >/dev/null 2>&1; then cygpath -u "$1"; else printf '%s\n' "$1"; fi
}
ROOT="$(norm "${CLAUDE_PROJECT_DIR:-$PWD}")"
MS="$HOME/.claude/memory-system"
GLOBAL="$HOME/.claude/memory-global"
STATE="$MS/state/$(printf %s "$ROOT" | { shasum 2>/dev/null || sha1sum; } | cut -c1-12)"
enabled() { [ -f "$ROOT/memory/INDEX.md" ]; }
# Add this project to the global registry (name<TAB>root) if missing.
register() {
  mkdir -p "$GLOBAL"; touch "$GLOBAL/projects.tsv"
  cut -f2 "$GLOBAL/projects.tsv" | grep -qxF "$ROOT" || printf '%s\t%s\n' "$(basename "$ROOT")" "$ROOT" >> "$GLOBAL/projects.tsv"
}
# If path $1 lies in another enabled project's memory/ (<D>/memory/ with memory/INDEX.md, D != ROOT), print D and succeed.
other_memory() {
  local F="$1" D
  case "$F" in /*) ;; *) F="$PWD/$F" ;; esac
  D=$(dirname "$F")
  while [ "$D" != "/" ] && [ -n "$D" ]; do
    if [ -f "$D/memory/INDEX.md" ]; then
      case "$F" in "$D/memory/"*) [ "$D" != "$ROOT" ] && { echo "$D"; return 0; } ;; esac
      return 1
    fi
    D=$(dirname "$D")
  done
  return 1
}
# Registered project roots other than this one.
others() { [ -f "$GLOBAL/projects.tsv" ] && cut -f2 "$GLOBAL/projects.tsv" | grep -vxF "$ROOT"; }
# History week name YYYY-Www, named after the Sunday that starts the week (date +%U), so the
# week around New Year stays in one file. $1 = weeks back (default 0). GNU or BSD date.
week_file() {
  local back=$(( $(date +%w) + 7 * ${1:-0} ))
  date -d "-$back days" +%Y-W%U 2>/dev/null || date -v-"$back"d +%Y-W%U
}
# History file for an entry ID: H-202610-0042 -> 2026-10.md, H-2026W42-0042 -> 2026-W42.md.
id_file() {
  case "$1" in
    H-[0-9][0-9][0-9][0-9]W[0-9][0-9]-*) printf '%s-W%s.md\n' "${1:2:4}" "${1:7:2}" ;;
    H-[0-9][0-9][0-9][0-9][0-9][0-9]-*) printf '%s-%s.md\n' "${1:2:4}" "${1:6:2}" ;;
    *) return 1 ;;
  esac
}
# The history file and entry ID logged for this turn (written by log-prompt.sh). Falls back to
# the newest history file and its last entry for turns logged before .turn-hist existed.
turn_hist() {
  TURN_FILE=""; TURN_ID=""
  [ -f "$STATE/.turn-hist" ] && IFS=$'\t' read -r TURN_FILE TURN_ID < "$STATE/.turn-hist"
  if [ ! -f "$TURN_FILE" ]; then
    TURN_FILE=$(ls -t "$ROOT"/memory/history/*.md 2>/dev/null | head -1)
    TURN_ID=""
  fi
  [ -n "$TURN_ID" ] || TURN_ID=$(grep -o '^### H-[0-9W-]*' "$TURN_FILE" 2>/dev/null | tail -1 | sed 's/^### //')
}
~~~~

### File 2: `~/.claude/memory-system/hooks/log-prompt.sh`

~~~~bash
#!/bin/bash
# UserPromptSubmit hook: start the turn and log the user's prompt to memory/history/YYYY-Www.md
# (one file per week, weeks start on Sunday). Also asks Claude once per week to sum up last
# week's entries that have no Did (anything printed to stdout is added to Claude's context).
. "$(dirname "$0")/lib.sh"
enabled || exit 0
register
mkdir -p "$STATE"
touch "$STATE/.turn-start"
: > "$STATE/.turn-edits"
rm -f "$STATE/.turn-logged" "$STATE/.turn-hist" "$STATE/.turn-lookup" "$STATE/.turn-weeksum"

PROMPT=$(jq -r '.prompt // empty' | tr '\n\r\t' '   ' | tr -s ' ' | cut -c1-300)
[ -z "$PROMPT" ] && exit 0

H="$ROOT/memory/history"
mkdir -p "$H"
WK=$(week_file)
F="$H/$WK.md"
[ -f "$F" ] || printf '# History %s\n\nOne entry per request: what was asked, what was done, files touched. Look up with ~/.claude/memory-system/lookup-memory.sh.\n' "$WK" > "$F"
# Next sequential ID for this week: H-YYYYWww-NNNN (highest existing + 1). The ID names its file.
P="H-${WK%-W*}W${WK#*-W}"
LAST=$(grep -o "^### $P-[0-9]*" "$F" | grep -o '[0-9]*$' | sort -n | tail -1)
ID=$(printf '%s-%04d' "$P" $((10#${LAST:-0} + 1)))
printf '\n### %s · %s\n**Asked:** %s\n' "$ID" "$(date '+%Y-%m-%d %H:%M')" "$PROMPT" >> "$F"
printf '%s\t%s\n' "$F" "$ID" > "$STATE/.turn-hist"

# Weekly summary: last week's file has entries without a Did but no "## Week summary" block yet.
PREV_WK=$(week_file 1)
PREV="$H/$PREV_WK.md"
if [ -f "$PREV" ] && ! grep -q '^## Week summary' "$PREV"; then
  LIST=$(LC_ALL=C awk '
    function flush() { if (id != "" && !did) { a = asked; if (length(a) > 120) { a = substr(a, 1, 120); sub(/[\200-\377]+$/, "", a); a = a "…" }; print "- " id ": " a } }
    /^### /  { flush(); id = $2; did = 0; asked = ""; next }
    /^## /   { flush(); id = ""; next }
    /^\*\*Asked:\*\*/ { asked = $0; sub(/^\*\*Asked:\*\* */, "", asked); next }
    /^\*\*Did:\*\*/   { did = 1 }
    END { flush() }' "$PREV" | head -80)
  if [ -n "$LIST" ]; then
    printf '%s\n' "$PREV" > "$STATE/.turn-weeksum"   # guard-history.sh lets Claude Read this file
    cat <<EOF
Memory housekeeping (do this first, then answer the request): last week's history file memory/history/$PREV_WK.md has entries without a Did and no week summary yet. Append these two lines at the very end of that file, and nothing else:

## Week summary
<one paragraph, 2-3 plain sentences on what the requests below were about, so they can be found later>

The requests without a Did (for you to summarise; do not copy them into the file):
$LIST
EOF
  fi
fi
exit 0
~~~~

### File 3: `~/.claude/memory-system/hooks/log-edit.sh`

~~~~bash
#!/bin/bash
# PostToolUse hook: record each file edited by Claude this turn.
. "$(dirname "$0")/lib.sh"
enabled || exit 0
mkdir -p "$STATE"
F=$(norm "$(jq -r '.tool_input.file_path // .tool_response.filePath // empty')")
[ -n "$F" ] && echo "$F" >> "$STATE/.turn-edits"
exit 0
~~~~

### File 4: `~/.claude/memory-system/hooks/check-memory.sh`

~~~~bash
#!/bin/bash
# Stop hook:
#  1. Block finishing a turn if Claude edited code this turn but memory/ was not updated, or
#     edited any file but this turn's history entry has no **Did:** line.
#     Only files edited via Claude's tools count (logged by log-edit.sh), so changes pulled
#     in from other people never trigger it.
#  2. Otherwise, show the user a one-line summary of what changed in memory/ this turn.

. "$(dirname "$0")/lib.sh"
HOOKS="$(dirname "$0")"
INPUT=$(cat)
enabled || exit 0
MARKER="$STATE/.turn-start"
EDITS="$STATE/.turn-edits"
ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active // false' 2>/dev/null)
turn_hist   # sets TURN_FILE (this turn's history file) and TURN_ID (its entry)
HIST="$TURN_FILE"

say() { jq -n --arg m "$1" '{systemMessage: $m}'; }
# git pinned to the memory/ repo, so it can never fall back to a parent (code) repo.
mgit() { GIT_DIR="$ROOT/memory/.git" GIT_WORK_TREE="$ROOT/memory" git -C "$ROOT/memory" "$@"; }
# Value of a **Label:** line inside this turn's entry (an entry ends at the next ### or ## heading).
entry_val() {
  awk -v id="$TURN_ID" -v lab="$1" '
    /^### / { in_e = (index($0, "### " id " ") == 1); next }
    /^## /  { in_e = 0; next }
    in_e && index($0, "**" lab ":**") == 1 { v = substr($0, length(lab) + 6); sub(/^ +/, "", v) }
    END { print v }' "$HIST" 2>/dev/null
}

# Memory files created/updated since the turn started (history and generated files excluded).
NOTES=""
if [ -f "$MARKER" ]; then
  NOTES=$(find "$ROOT/memory" \( -name .git -o -name history -o -path "$ROOT/memory/refs" \) -prune -o -type f ! -name REFS.md -newer "$MARKER" -print 2>/dev/null)
fi

summary() {
  local out="" f rel verb title nid tags what did htags
  if [ -n "$NOTES" ]; then
    while IFS= read -r f; do
      rel="${f#$ROOT/memory/}"
      if mgit ls-files --error-unmatch "$rel" >/dev/null 2>&1; then verb="Updated"; else verb="Created"; fi
      title=$(grep -m1 '^# ' "$f" 2>/dev/null | sed 's/^# *//')
      nid=$(grep -m1 '^ID:' "$f" 2>/dev/null | sed 's/^ID:[[:space:]]*//')
      tags=$(grep -m1 '^Tags:' "$f" 2>/dev/null | sed 's/^Tags:[[:space:]]*//')
      what=$(grep -m1 '^Last change:' "$f" 2>/dev/null | sed 's/^Last change:[[:space:]]*//' | cut -c1-250)
      out="$out"$'\n'"  • $verb $rel"
      [ -n "$title" ] && out="$out — $title"
      [ -n "$nid$tags" ] && out="$out  [$nid $tags]"
      [ -n "$what" ] && out="$out"$'\n'"      $what"
    done <<< "$NOTES"
  fi
  # This turn's history entry (written by the hook, completed by Claude).
  if [ -f "$HIST" ] && [ -f "$MARKER" ] && [ "$HIST" -nt "$MARKER" ]; then
    did=$(entry_val Did | cut -c1-250)
    htags=$(entry_val Tags)
    if [ -n "$did" ]; then
      out="$out"$'\n'"  • History $TURN_ID  $htags"$'\n'"      $did"
    fi
  fi
  if [ -z "$out" ]; then
    say "memory: no changes made"
  else
    say "memory updated:$out"
  fi
}

# Block (skip on the forced second pass):
#  - code was edited but no note was touched;
#  - any file was edited (anywhere, except history, generated lookup files and ~/.claude/plans/)
#    but this turn's entry has no Did.
if [ "$ACTIVE" != "true" ] && [ -f "$MARKER" ] && [ -s "$EDITS" ]; then
  MSG=""
  if [ -z "$NOTES" ]; then
    CHANGED=$(sort -u "$EDITS" | grep -E "^$ROOT/" | grep -vE "^$ROOT/(memory|\.claude)/|/(\.git|\.gradle|\.idea|build|node_modules)/|/CLAUDE\.md$" | head -20)
    [ -n "$CHANGED" ] && MSG="You edited code this turn but nothing in memory/ was updated. Per CLAUDE.md, create or update the matching note now. Edited files:"$'\n'"$CHANGED"$'\n'
  fi
  # Plan files (~/.claude/plans/) don't count: in plan mode they are the only editable file.
  EDITED=$(sort -u "$EDITS" | grep -vE "^$ROOT/memory/(history/|refs/|REFS\.md$)" | grep -vF "$HOME/.claude/plans/" | head -20)
  if [ -n "$EDITED" ] && [ -f "$HIST" ] && [ -z "$(entry_val Did)" ]; then
    MSG="$MSG""You edited files this turn but history entry $TURN_ID has no Did. Add **Did:** (1-2 sentences) and **Tags:** directly under its **Asked:** line in memory/history/$(basename "$HIST"). Edited files:"$'\n'"$EDITED"
  fi
  if [ -n "$MSG" ]; then
    printf '%s\n' "$MSG" >&2
    exit 2
  fi
fi

# Append the files touched this turn to the history entry (once per turn).
if [ -f "$HIST" ] && [ ! -f "$STATE/.turn-logged" ] && [ -s "$EDITS" ]; then
  FILES=$(sort -u "$EDITS" | grep -E "^$ROOT/" | grep -vE "^$ROOT/(\.claude)/|/CLAUDE\.md$" | sed "s|^$ROOT/||" | head -15 | paste -sd, - | sed 's/,/, /g')
  [ -n "$FILES" ] && printf '**Files:** %s\n' "$FILES" >> "$HIST"
  touch "$STATE/.turn-logged"
fi

"$HOOKS/build-refs.sh" 2>/dev/null
"$HOOKS/build-global-index.sh" 2>/dev/null

# Commit memory/ changes (the only place anything is committed). Runs after summary(),
# which needs the pre-commit state for Created/Updated. Never inits, branches or pushes;
# failures are silent and leftovers go into the next turn's commit.
commit_memory() {
  [ -d "$ROOT/memory/.git" ] || return 0
  [ "$(mgit rev-parse --show-toplevel 2>/dev/null)" = "$(cd "$ROOT/memory" && pwd -P)" ] || return 0
  [ -n "$(mgit status --porcelain 2>/dev/null)" ] || return 0
  mgit add -A >/dev/null 2>&1 &&
    mgit -c commit.gpgsign=false commit -q --no-verify \
      -m "memory: ${TURN_ID:+$TURN_ID }$(date '+%Y-%m-%d %H:%M')" >/dev/null 2>&1
  return 0
}

summary
commit_memory
exit 0
~~~~

### File 5: `~/.claude/memory-system/hooks/build-refs.sh`

~~~~bash
#!/bin/bash
# Regenerate the lookup files from history and notes. Never edit them by hand.
#   memory/REFS.md          small map, read in full: notes, tags, months
#   memory/refs/YYYY-MM.md  one per month: entries that have a Did, tag lists, and one line per
#                           history file for the entries without a Did (with its week summary)
# A file is only rewritten when its content changes, so past months keep their mtime.
. "$(dirname "$0")/lib.sh"
enabled || exit 0
# Byte-based and identical on macOS and Linux: same sort order, same cuts on every machine.
export LC_ALL=C
MEM="$ROOT/memory"
REFS_DIR="$MEM/refs"
TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT
ENT="$TMP/entries.tsv"   # id  month  date  tags  did  refs        (entries that have a Did)
NOD="$TMP/nodid.tsv"     # file  month  count  summary              (one line per history file)
NOT="$TMP/notes.tsv"     # folder  rel  id  status  updated  tags  title  summary
MEAN="$TMP/tags.tsv"     # tag  meaning                              (from TAGS.md)
: > "$ENT"; : > "$NOD"; : > "$NOT"; : > "$MEAN"
CUR_WK=$(week_file)

# Replace $2 with $1 only if the content differs.
put() { cmp -s "$1" "$2" 2>/dev/null || { mkdir -p "$(dirname "$2")"; cp "$1" "$2"; }; }

# History (weekly YYYY-Www.md and legacy monthly YYYY-MM.md). An entry ends at the next ### or ##.
for f in $(ls "$MEM"/history/*.md 2>/dev/null | sort); do
  awk -v OFS='\t' -v fn="$(basename "$f" .md)" -v E="$ENT" -v N="$NOD" '
    function clean(s) { gsub(/\t/, " ", s); sub(/ +$/, "", s); return s }
    function val(  v) { v = $0; sub(/^\*\*[A-Za-z]+:\*\* */, "", v); return clean(v) }
    function flush() {
      if (id == "") return
      if (first == "") first = substr(date, 1, 7)
      if (did != "") print id, substr(date, 1, 7), date, tags, did, refs >> E
      else nod++
      id = ""
    }
    /^### / {
      flush(); h = substr($0, 5); id = h; sub(/ .*/, "", id); date = h; sub(/^[^·]*· */, "", date)
      if (id !~ /^H-/) { id = "H-legacy"; date = h }
      did = tags = refs = ""; insum = 0; next }
    /^## / { flush(); insum = ($0 ~ /^## Week summary/ && sum == ""); next }
    insum { if ($0 ~ /^[[:space:]]*$/) { if (sum != "") insum = 0 } else sum = sum (sum == "" ? "" : " ") clean($0); next }
    /^\*\*Did:\*\*/  { did = val(); next }
    /^\*\*Tags:\*\*/ { tags = val(); next }
    /^\*\*Refs:\*\*/ { refs = val(); next }
    END { flush(); if (first != "") print fn, first, nod + 0, sum >> N }' "$f"
done

# Notes: title, ID, status, updated, tags and a one-line summary (Summary: line, else the first
# line of "## Goal / Context").
find "$MEM" \( -name .git -o -name history -o -path "$REFS_DIR" \) -prune -o -type f -name '*.md' -print 2>/dev/null | sort |
while IFS= read -r f; do
  b=$(basename "$f" .md); case "$b" in INDEX|TAGS|REFS) continue ;; esac
  rel="${f#$MEM/}"; dir=$(dirname "$rel"); [ "$dir" = . ] && dir="(top)"
  awk -v OFS='\t' -v rel="$rel" -v dir="$dir" -v b="$b" '
    function clean(s) { gsub(/\t/, " ", s); sub(/ +$/, "", s); return s }
    t == "" && /^# /       { t = clean(substr($0, 3)); next }
    id == "" && /^ID:/     { id = $0; sub(/^ID:[[:space:]]*/, "", id); id = clean(id); next }
    tg == "" && /^Tags:/   { tg = $0; sub(/^Tags:[[:space:]]*/, "", tg); tg = clean(tg); next }
    sm == "" && /^Summary:/ { sm = $0; sub(/^Summary:[[:space:]]*/, "", sm); sm = clean(sm); next }
    st == "" && /^Status:/ {
      st = $0; sub(/^Status:[[:space:]]*/, "", st); up = st
      sub(/[[:space:]]*Last updated:.*/, "", st)
      if (up ~ /Last updated:/) { sub(/.*Last updated:[[:space:]]*/, "", up); up = clean(up) } else up = ""
      st = clean(st); next }
    up == "" && /^Last updated:/ { up = $0; sub(/^Last updated:[[:space:]]*/, "", up); up = clean(up); next }
    /^## / { goal = ($0 ~ /^## Goal/); next }
    goal && gl == "" && NF { gl = clean($0) }
    END {
      if (id == "") id = "N-" b
      if (sm == "") { sm = gl; if (length(sm) > 100) { sm = substr(sm, 1, 99); sub(/[\200-\377]+$/, "", sm); sm = sm "…" } }
      print dir, rel, id, st, up, tg, t, sm }' "$f" >> "$NOT"
done

grep '^- #' "$MEM/TAGS.md" 2>/dev/null | sed -E 's/^- (#[^[:space:]]+)[[:space:]]*(—|-)?[[:space:]]*/\1	/' > "$MEAN"

ENTRY_FMT='An ID names its history file: H-2026W42-0042 → history/2026-W42.md.'

# refs/YYYY-MM.md, one per month that has entries.
MONTHS=$( { cut -f2 "$ENT"; cut -f2 "$NOD"; } | grep -E '^[0-9]{4}-[0-9]{2}$' | sort -ru)
for m in $MONTHS; do
  out="$TMP/refs-$m.md"
  {
    echo "# Refs $m (generated — do not edit)"
    echo
    echo "Entries that have a Did, newest first: \`ID | date | tags | Did | Refs\`. $ENTRY_FMT"
    echo
    echo "## Entries"
    echo
    awk -F'\t' -v m="$m" '$2 == m' "$ENT" | sort -t$'\t' -k3,3r -k1,1r |
      awk -F'\t' '{ d = $5; if (length(d) > 140) { d = substr(d, 1, 139); sub(/[\200-\377]+$/, "", d); d = d "…" }
                    printf "- %s | %s | %s | %s%s\n", $1, $3, ($4 == "" ? "#untagged" : $4), d, ($6 == "" ? "" : " | " $6) }'
    echo
    echo "## By tag"
    echo
    awk -F'\t' -v m="$m" '$2 == m' "$ENT" | sort -t$'\t' -k3,3r -k1,1r |
      awk -F'\t' '{ n = split($4, a, /[ ,]+/); for (i = 1; i <= n; i++) if (a[i] ~ /^#/) l[a[i]] = (a[i] in l ? l[a[i]] ", " : "") $1 }
                  END { for (t in l) print "- " t ": " l[t] }' | sort
    echo
    echo "## Without a Did (lookup-memory.sh searches these too)"
    echo
    awk -F'\t' -v m="$m" -v cw="$CUR_WK" '$2 == m && $3 > 0 {
        s = $4
        if (s == "") s = ($1 == cw ? "summary at week end" : ($1 ~ /-W[0-9][0-9]$/ ? "no week summary yet" : "old monthly file, no week summary"))
        printf "- %s | %d entries | history/%s.md | %s\n", $1, $3, $1, s }' "$NOD" | sort -r
  } > "$out"
  put "$out" "$REFS_DIR/$m.md"
done

# Top-level REFS.md: notes, tags, months.
{
  echo "# REFS (generated — do not edit)"
  echo
  echo "How to look up past work:"
  echo "1. Run \`bash ~/.claude/memory-system/lookup-memory.sh\` (no args = this turn's question; or words, --when, --tag)."
  echo "   It searches the notes below, every history entry (also those without a Did) and the week summaries."
  echo "2. Open the notes it lists; full entries with \`lookup-memory.sh --show <ID>\`. $ENTRY_FMT"
  echo "3. Only then, if needed: grep history/*.md (blocked by a hook until the script has run this turn)."
  echo "If nothing matches, say the lookup missed."
  echo
  echo "## Notes"
  sort -t$'\t' -k1,1 -k2,2 "$NOT" | awk -F'\t' '
    $1 != last { printf "\n### %s/\n\n", $1; last = $1 }
    { printf "- %s | %s | %s | %s | %s%s\n", $3, ($4 == "" ? "?" : $4), ($5 == "" ? "?" : $5), ($6 == "" ? "#untagged" : $6), $7, ($8 == "" ? "" : " — " $8) }'
  echo
  echo "## Tags"
  echo
  awk -F'\t' '
    FILENAME == ARGV[1] { mean[$1] = $2; known[$1] = 1; all[$1] = 1; next }
    FILENAME == ARGV[2] { n = split($6, a, /[ ,]+/); for (i = 1; i <= n; i++) if (a[i] ~ /^#/) { all[a[i]] = 1; nl[a[i]] = (a[i] in nl ? nl[a[i]] ", " : "") $3 }; next }
    { n = split($4, a, /[ ,]+/); for (i = 1; i <= n; i++) if (a[i] ~ /^#/) { all[a[i]] = 1; c[a[i] SUBSEP $2]++; if (!((a[i], $2) in seen)) { seen[a[i], $2] = 1; ms[a[i]] = ms[a[i]] " " $2 } } }
    END {
      for (t in all) {
        line = "- " t (t in known ? (mean[t] == "" ? "" : " — " mean[t]) : " (not in TAGS.md)")
        if (t in nl) line = line " | notes: " nl[t]
        k = split(ms[t], mm, " "); out = ""
        for (i = 1; i <= k; i++) for (j = i + 1; j <= k; j++) if (mm[j] > mm[i]) { x = mm[i]; mm[i] = mm[j]; mm[j] = x }
        for (i = 1; i <= k; i++) out = out (out == "" ? "" : ", ") mm[i] " (" c[t SUBSEP mm[i]] ")"
        if (out != "") line = line " | months: " out
        if (!(t in nl) && out == "") line = line " | no uses yet"
        print line
      }
    }' "$MEAN" "$NOT" "$ENT" | sort
  echo
  echo "## Months"
  echo
  for m in $MONTHS; do
    d=$(awk -F'\t' -v m="$m" '$2 == m' "$ENT" | wc -l | tr -d ' ')
    n=$(awk -F'\t' -v m="$m" '$2 == m { s += $3 } END { print s + 0 }' "$NOD")
    echo "- $m | $d with a Did, $n without | refs/$m.md"
  done
} > "$TMP/REFS.md"
put "$TMP/REFS.md" "$MEM/REFS.md"
exit 0
~~~~

### File 6: `~/.claude/memory-system/hooks/build-global-index.sh`

~~~~bash
#!/bin/bash
# Regenerate ~/.claude/memory-global/GLOBAL_INDEX.md: one line per note across all registered projects.
GLOBAL="$HOME/.claude/memory-global"
REG="$GLOBAL/projects.tsv"
[ -f "$REG" ] || exit 0
OUT="$GLOBAL/GLOBAL_INDEX.md"
TMP=$(mktemp); trap 'rm -f "$TMP"' EXIT
{
  echo "# GLOBAL INDEX (generated — do not edit)"
  echo
  echo "One line per note across projects: \`project | note ID | path | tags | last change\`. For request history open the project's own \`memory/REFS.md\`."
  while IFS=$'\t' read -r name root; do
    [ -d "$root/memory" ] || continue
    echo; echo "## $name"; echo "Root: $root"; echo
    find "$root/memory" \( -name .git -o -name history -o -path "$root/memory/refs" \) -prune -o -type f -name '*.md' -print 2>/dev/null | sort | while read -r f; do
      b=$(basename "$f" .md); case "$b" in INDEX|TAGS|REFS) continue ;; esac
      nid=$(grep -m1 '^ID:' "$f" | sed 's/^ID:[[:space:]]*//'); [ -z "$nid" ] && nid="N-$b"
      tg=$(grep -m1 '^Tags:' "$f" | sed 's/^Tags:[[:space:]]*//')
      lc=$(grep -m1 '^Last change:' "$f" | sed 's/^Last change:[[:space:]]*//' | cut -c1-200)
      printf -- '- %s | %s | %s | %s | %s\n' "$name" "$nid" "${f#$root/memory/}" "${tg:-#untagged}" "$lc"
    done
  done < "$REG"
} > "$TMP" && mv "$TMP" "$OUT"
exit 0
~~~~

### File 7: `~/.claude/memory-system/hooks/session-status.sh`

~~~~bash
#!/bin/bash
# SessionStart hook: always show whether memory is enabled for this project.
. "$(dirname "$0")/lib.sh"
# Drop stale cross-project read unlocks (guard-memory-read.sh).
find "$MS/state" -name 'allow-cross-*' -mtime +1 -delete 2>/dev/null
if enabled; then
  register
  N=$(find "$ROOT/memory" \( -name .git -o -name history -o -path "$ROOT/memory/refs" \) -prune -o -type f -name '*.md' ! -name INDEX.md ! -name TAGS.md ! -name REFS.md -print 2>/dev/null | wc -l | tr -d ' ')
  L=$(grep -ho '^### H-[0-9W-]*' "$ROOT/memory/history/"*.md 2>/dev/null | sort | tail -1 | sed 's/^### //')
  M="memory: enabled — $N notes, last entry ${L:-none} — run ~/.claude/memory-system/init-memory.sh --disable to disable"
else
  M="memory: not enabled here — run ~/.claude/memory-system/init-memory.sh to enable"
fi
jq -n --arg m "$M" '{systemMessage: $m}'
exit 0
~~~~

### File 8: `~/.claude/memory-system/hooks/guard-memory.sh`

~~~~bash
#!/bin/bash
# PreToolUse hook (Write|Edit|NotebookEdit): block writes into another project's memory/.
# A path counts as "another project's memory" if it lies under <D>/memory/ where D has memory/INDEX.md and D != this project.
. "$(dirname "$0")/lib.sh"
F=$(norm "$(jq -r '.tool_input.file_path // .tool_input.notebook_path // empty')")
[ -z "$F" ] && exit 0
if D=$(other_memory "$F"); then
  echo "Blocked: $F is in another project's memory ($D). Memory may only be written in the current project ($ROOT)." >&2
  exit 2
fi
exit 0
~~~~

### File 9: `~/.claude/memory-system/hooks/guard-memory-read.sh`

~~~~bash
#!/bin/bash
# Cross-project memory read guard (Read|Grep|Glob|Bash).
#   PreToolUse:                 ask before reading another project's memory/ or memory-global/, unless unlocked this session.
#   PostToolUse (--approved):   the tool ran, so the user approved; unlock cross-project memory reads for this session.
. "$(dirname "$0")/lib.sh"
IN=$(cat)
SID=$(printf %s "$IN" | jq -r '.session_id // empty')
TOOL=$(printf %s "$IN" | jq -r '.tool_name // empty')
FLAG="$STATE/allow-cross-${SID:-none}"
# Lookup-order gate for this project's history (exit 3 = block; any other status, even a crash, allows).
printf %s "$IN" | bash "$(dirname "$0")/guard-history.sh" "$1"
[ $? = 3 ] && exit 2

# Is path $1 a guarded memory location? For searches ($2=search), also an ancestor of another project.
guarded() {
  local P="$1"
  case "$P" in "~"*) P="$HOME${P#\~}" ;; /*) ;; *) P="$PWD/$P" ;; esac
  P="${P%/}"
  case "$P/" in "$GLOBAL/"*) return 0 ;; esac
  other_memory "$P/" >/dev/null && return 0
  if [ "$2" = search ]; then
    local R
    while IFS= read -r R; do
      [ -n "$R" ] || continue
      # Grep/Glob honor .gitignore, so a git-ignored memory/ is not searched.
      git -C "$R" check-ignore -q "$R/memory/" 2>/dev/null && continue
      case "$R/" in "$P/"*) return 0 ;; esac
    done < <(others)
  fi
  return 1
}

TARGET=""
case "$TOOL" in
  Read)
    F=$(norm "$(printf %s "$IN" | jq -r '.tool_input.file_path // empty')")
    [ -n "$F" ] && guarded "$F" && TARGET="$F" ;;
  Grep|Glob)
    F=$(norm "$(printf %s "$IN" | jq -r '.tool_input.path // empty')")
    G=$(printf %s "$IN" | jq -r '.tool_input.pattern // empty')
    if guarded "${F:-$PWD}" search; then TARGET="${F:-$PWD}"
    elif [ "$TOOL" = Glob ] && case "$G" in /*|"~"*) true ;; *) false ;; esac && guarded "${G%%[*?[{]*}" search; then TARGET="$G"
    fi ;;
  Bash)
    C=$(printf %s "$IN" | jq -r '.tool_input.command // empty')
    case "$C" in *memory-global*) TARGET="memory-global (Bash)" ;; esac
    if [ -z "$TARGET" ]; then
      while IFS= read -r R; do
        [ -n "$R" ] || continue
        T="~${R#$HOME}"
        case "$C" in *"$R/memory"*|*"$T/memory"*) TARGET="$R/memory (Bash)"; break ;; esac
      done < <(others)
    fi ;;
esac
[ -z "$TARGET" ] && exit 0

if [ "$1" = --approved ]; then
  [ -n "$SID" ] && { mkdir -p "$STATE"; touch "$FLAG"; }
  exit 0
fi
[ -n "$SID" ] && [ -f "$FLAG" ] && exit 0
jq -n --arg r "Cross-project memory read: $TARGET. Current project: $ROOT. Approving unlocks cross-project memory reads for this session." \
  '{hookSpecificOutput:{hookEventName:"PreToolUse",permissionDecision:"ask",permissionDecisionReason:$r}}'
exit 0
~~~~

### File 10: `~/.claude/memory-system/hooks/guard-history.sh`

~~~~bash
#!/bin/bash
# Lookup-order gate for this project's memory/history/ (Read|Grep|Bash). Called by
# guard-memory-read.sh with the hook JSON on stdin and its mode in $1:
#   PreToolUse:               block grepping/reading history until lookup-memory.sh ran this turn
#   PostToolUse (--approved): a Bash call ran lookup-memory.sh, so unlock history for this turn
# Exit 3 means "block" (the caller turns it into exit 2 with this script's stderr); any other
# status allows, so a crash or syntax error here never blocks tools. log-prompt.sh re-arms the
# gate each turn by removing $STATE/.turn-lookup.
. "$(dirname "$0")/lib.sh"
enabled || exit 0
IN=$(cat)
TOOL=$(printf %s "$IN" | jq -r '.tool_name // empty')
CMD=""; [ "$TOOL" = Bash ] && CMD=$(printf %s "$IN" | jq -r '.tool_input.command // empty')

if [ "$1" = --approved ]; then
  case "$CMD" in *lookup-memory.sh*) mkdir -p "$STATE" && touch "$STATE/.turn-lookup" ;; esac
  exit 0
fi
[ -f "$STATE/.turn-lookup" ] && exit 0

M="$ROOT/memory"
H="$M/history"
CWD=$(norm "$(printf %s "$IN" | jq -r '.cwd // empty')"); [ -n "$CWD" ] || CWD="$PWD"
abs() {
  local P; P=$(norm "$1")
  case "$P" in "~"*) printf %s "$HOME${P#\~}" ;; /*) printf %s "$P" ;; *) printf %s "$CWD/$P" ;; esac
}
# ERE-escaped string.
ere() { printf %s "$1" | sed 's/[].[^$*+?(){}|\\/]/\\&/g'; }

block() {
  cat >&2 <<EOF
Memory lookup order: run \`bash ~/.claude/memory-system/lookup-memory.sh\` first (no args = this turn's question, or pass words / --when today|last-week|YYYY-MM-DD / --tag '#x'; --show <ID> prints full entries). It searches notes, refs and every history entry, including those without a Did. Grep/Read of memory/history/ is unlocked for this turn once it has run. ($1) To search notes only, narrow the path to a note folder such as memory/system/.
EOF
  exit 3
}

case "$TOOL" in
  Grep)
    P=$(printf %s "$IN" | jq -r '.tool_input.path // empty'); P=$(abs "${P:-$CWD}"); P="${P%/}"
    G=$(printf %s "$IN" | jq -r '.tool_input.glob // empty')
    case "$P/" in "$H/"*) block "Grep on history" ;; esac
    [ "$P" = "$M" ] && block "Grep on memory/ includes history"
    case "$G" in *history*) { [ "$P" = "$ROOT" ] || [ "$P" = "$M" ]; } && block "Grep with a history glob" ;; esac
    ;;
  Read)
    F=$(abs "$(printf %s "$IN" | jq -r '.tool_input.file_path // empty')")
    case "$F" in "$H/"*) ;; *) exit 0 ;; esac
    # Allowed: last week's file when log-prompt.sh asked for its week summary this turn, and this
    # turn's file once something was edited (the Did is required only then; Edit needs a Read).
    [ -f "$STATE/.turn-weeksum" ] && [ "$F" = "$(cat "$STATE/.turn-weeksum")" ] && exit 0
    turn_hist
    [ "$F" = "$TURN_FILE" ] && [ -s "$STATE/.turn-edits" ] && exit 0
    block "Read of a history file"
    ;;
  Bash)
    [ -n "$CMD" ] || exit 0
    case "$CMD" in *lookup-memory.sh*) exit 0 ;; esac
    printf %s "$CMD" | grep -qE '(^|[^a-zA-Z0-9_-])(grep|egrep|fgrep|rg|ag|ack|cat|sed|awk|head|tail|less|more|find|bat)([^a-zA-Z0-9_-]|$)' || exit 0
    E=$(ere "$M"); ET=$(ere "~${M#$HOME}")
    HIST=0; REC=0; INMEM=0; ABS=0; REL=0; ROOTREF=0
    printf %s "$CMD" | grep -q history && HIST=1
    printf %s "$CMD" | grep -qE '(^|[^a-zA-Z0-9_-])(rg|ag|ack|find)([^a-zA-Z0-9_-]|$)|grep[^|;&]*[[:space:]]-[a-zA-Z]*[rR]' && REC=1
    case "$CWD/" in "$M/"*) INMEM=1 ;; esac
    printf %s "$CMD" | grep -qE '(^|[;&|[:space:]])cd[[:space:]]+[^;&|]*memory(/history)?/?([[:space:]]|[;&|]|$)' && INMEM=1
    printf %s "$CMD" | grep -qE "($E|$ET)" && ABS=1
    printf %s "$CMD" | grep -qE "(^|[[:space:]'\"=(])(\\./)?memory(/|[[:space:]'\"]|\$)" && REL=1
    # memory/ itself (not a subfolder) as a search target.
    printf %s "$CMD" | grep -qE "($E|$ET|(^|[[:space:]'\"=(])(\\./)?memory)/?([[:space:]'\"]|\$)" && ROOTREF=1
    # Searches of the hook scripts (~/.claude) that only mention memory/history as text pass.
    case "$CMD" in *"/.claude"*) [ $ABS = 0 ] && [ $INMEM = 0 ] && exit 0 ;; esac
    if [ $HIST = 1 ] && [ $((ABS + REL + INMEM)) -gt 0 ]; then block "Bash read/search of history"; fi
    if [ $REC = 1 ] && { [ $ROOTREF = 1 ] || [ $INMEM = 1 ]; }; then block "recursive search over memory/ includes history"; fi
    ;;
esac
exit 0
~~~~

### File 11: `~/.claude/memory-system/lookup-memory.sh`

~~~~bash
#!/bin/bash
# Look up past work in the current project's memory/ (notes, refs and every history entry,
# including entries without a Did). Run this before grepping or reading memory/history/:
# guard-history.sh blocks those until this script has run in the current turn.
#   lookup-memory.sh                               this turn's question (the hook-logged Asked)
#   lookup-memory.sh [--when W] [--tag T] [words]  W: today|yesterday|this-week|last-week|this-month|
#                                                  last-month|YYYY-MM-DD|YYYY-MM; T: #tag (repeatable)
#   lookup-memory.sh --show H-2026W41-0004 [N-…]   full history entries / notes by ID
#   lookup-memory.sh --all …                       no output caps
# Time words (today, last week, 2026-10-15, october, …), #tags and IDs inside the words are
# picked up too. Always exits 0.
export LC_ALL=C

# Project root: CLAUDE_PROJECT_DIR when it is an enabled project (set for hooks, not in Bash
# commands), else the nearest directory at or above $PWD with memory/INDEX.md. Windows paths
# (C:\...) are converted with cygpath when it exists.
R=""
P0=""; [ -n "$CLAUDE_PROJECT_DIR" ] && P0=$(cygpath -u "$CLAUDE_PROJECT_DIR" 2>/dev/null || printf %s "$CLAUDE_PROJECT_DIR")
if [ -n "$P0" ] && [ -f "$P0/memory/INDEX.md" ]; then
  R="$P0"
else
  D="$PWD"
  while [ -n "$D" ]; do
    [ -f "$D/memory/INDEX.md" ] && { R="$D"; break; }
    [ "$D" = / ] && break
    D=$(dirname "$D")
  done
fi
[ -n "$R" ] || { echo "lookup: memory not enabled here (no memory/INDEX.md at or above $PWD)"; exit 0; }
# lib.sh derives ROOT and STATE from CLAUDE_PROJECT_DIR, so the flag lands where the hooks look.
export CLAUDE_PROJECT_DIR="$R"
. "$HOME/.claude/memory-system/hooks/lib.sh"
MEM="$ROOT/memory"
mkdir -p "$STATE" && touch "$STATE/.turn-lookup"   # unlocks history reads for this turn
turn_hist   # TURN_FILE / TURN_ID: this turn's entry, left out of the results

SHOW=0; ALL=0; WHEN=""; TAGS=""; TEXT=""
while [ $# -gt 0 ]; do
  case "$1" in
    --show) SHOW=1 ;;
    --all) ALL=1 ;;
    --when) WHEN="$2"; shift ;;
    --when=*) WHEN="${1#--when=}" ;;
    --tag) TAGS="$TAGS $2"; shift ;;
    --tag=*) TAGS="$TAGS ${1#--tag=}" ;;
    -h|--help) sed -n '2,11p' "$0" | sed 's/^# \{0,1\}//'; exit 0 ;;
    *) TEXT="$TEXT $1" ;;
  esac
  shift
done
# No words, window or tags: use this turn's Asked as written by log-prompt.sh.
if [ $SHOW = 0 ] && [ -z "$TEXT$WHEN$TAGS" ]; then
  TEXT=$(awk -v id="$TURN_ID" '/^### / { in_e = (index($0, "### " id " ") == 1); next }
    in_e && /^\*\*Asked:\*\*/ { sub(/^\*\*Asked:\*\* */, ""); print; exit }' "$TURN_FILE" 2>/dev/null)
  [ -n "$TEXT" ] || { echo "lookup: no question given and no history entry for this turn. Pass words, --when or --tag."; exit 0; }
  echo "Query (this turn's Asked): $TEXT"
fi

# Print history entries / notes by ID.
show_ids() {
  local id f n
  for id in "$@"; do
    case "$id" in
      H-*)
        f="$MEM/history/$(id_file "$id")"
        if [ -f "$f" ] && grep -q "^### $id " "$f"; then
          echo "--- $id (history/$(basename "$f"))"
          awk -v id="$id" '/^### / { p = (index($0, "### " id " ") == 1) } /^## / { p = 0 } p && NF' "$f"
        else echo "--- $id: not found"; fi ;;
      N-*)
        n=$(find "$MEM" \( -name .git -o -name history -o -name refs \) -prune -o -type f -name "${id#N-}.md" -print 2>/dev/null | head -1)
        if [ -n "$n" ]; then echo "--- $id (${n#$ROOT/})"; cat "$n"; else echo "--- $id: not found"; fi ;;
    esac
    echo
  done
}
IDS=$(printf '%s\n' "$TEXT" | grep -oE 'H-[0-9]{6}-[0-9]+|H-[0-9]{4}W[0-9]{2}-[0-9]+|N-[a-z0-9][a-z0-9-]*[a-z0-9]' | awk '!s[$0]++')
if [ $SHOW = 1 ] || { [ -n "$IDS" ] && [ -z "$WHEN$TAGS" ]; }; then
  [ -n "$IDS" ] || { echo "lookup --show: give one or more IDs (H-… or N-…)"; exit 0; }
  # shellcheck disable=SC2086
  show_ids $IDS
  exit 0
fi

# Date window [FROM, TO] as YYYY-MM-DD strings (compared as text). GNU or BSD date.
dback() { date -d "-$1 days" +%F 2>/dev/null || date -v-"$1"d +%F; }
FROM=""; TO=""
widen() { # $1 from, $2 to
  { [ -z "$FROM" ] || [[ "$1" < "$FROM" ]]; } && FROM="$1"
  { [ -z "$TO" ] || [[ "$2" > "$TO" ]]; } && TO="$2"
}
window() {
  local w=$(date +%w) m
  case "$1" in
    today) widen "$(date +%F)" "$(date +%F)" ;;
    yesterday) widen "$(dback 1)" "$(dback 1)" ;;
    this-week) widen "$(dback "$w")" "$(date +%F)" ;;
    last-week) widen "$(dback $((w + 7)))" "$(dback $((w + 1)))" ;;
    this-month) widen "$(date +%Y-%m)-01" "$(date +%Y-%m)-31" ;;
    last-month) m=$(date -d "$(date +%Y-%m-01) -1 month" +%Y-%m 2>/dev/null || date -v1d -v-1m +%Y-%m); widen "$m-01" "$m-31" ;;
    [0-9][0-9][0-9][0-9]-[0-1][0-9]-[0-3][0-9]) widen "$1" "$1" ;;
    [0-9][0-9][0-9][0-9]-[0-1][0-9]) widen "$1-01" "$1-31" ;;
    *) echo "lookup: unknown --when '$1' (ignored)" ;;
  esac
}
# Month name -> YYYY-MM, the latest such month not in the future.
month_of() {
  local n y=$(date +%Y) cur=$((10#$(date +%m)))
  case "$1" in
    jan|january) n=1 ;; feb|february) n=2 ;; mar|march) n=3 ;; apr|april) n=4 ;; may) n=5 ;; jun|june) n=6 ;;
    jul|july) n=7 ;; aug|august) n=8 ;; sep|sept|september) n=9 ;; oct|october) n=10 ;; nov|november) n=11 ;; dec|december) n=12 ;;
    *) return 1 ;;
  esac
  [ "$n" -gt "$cur" ] && y=$((y - 1))
  printf '%s-%02d\n' "$y" "$n"
}
[ -n "$WHEN" ] && window "$WHEN"

STOP=" a an the what whats was were is are am be been being our ours we us you your i me my it its this that these those
 did do does done doing when where which who whom how why of to in on at for from with by and or but about into over
 any all some there here then than so if not no yes can could should would will shall may might must have has had
 please show tell find list give get got see look lookup search remember recall last past previous next ago
 today yesterday week weeks month months day days year time times ever before after earlier later again
 create created creating creates make made making add added adding update updated updating change changed changes
 work worked working thing things stuff something one also just only very really
 jan feb mar apr jun jul aug sep sept oct nov dec january february march april june july august september october
 november december "
WORDS=""
prev=""
for t in $(printf '%s\n' "$TEXT" | tr 'A-Z' 'a-z' | tr -c 'a-z0-9#\n-' ' '); do
  t="${t#-}"; t="${t%-}"
  case "$t" in
    today|yesterday) window "$t" ;;
    week) case "$prev" in this|current) window this-week ;; last|past|previous) window last-week ;; esac ;;
    month) case "$prev" in this|current) window this-month ;; last|past|previous) window last-month ;; esac ;;
    [0-9][0-9][0-9][0-9]-[0-1][0-9]-[0-3][0-9]|[0-9][0-9][0-9][0-9]-[0-1][0-9]) window "$t" ;;
    may|mar|march) case "$prev" in in|on|last|during|since|of) window "$(month_of "$t")" ;; esac ;;
    jan|january|feb|february|apr|april|jun|june|jul|july|aug|august|sep|sept|september|oct|october|nov|november|dec|december)
      window "$(month_of "$t")" ;;
    \#[a-z]*) TAGS="$TAGS $t" ;;
    h-[0-9]*|n-*) ;;
    *)
      case "$STOP" in *" $t "*|*" $t"$'\n'*) ;; *)
        if [ ${#t} -ge 3 ] && [ -n "${t//[0-9-]/}" ]; then
          if [ ${#t} -gt 4 ]; then
            case "$t" in *ing) t="${t%ing}" ;; *ed) t="${t%ed}" ;; *es) t="${t%es}" ;; *s) t="${t%s}" ;; esac
          fi
          case " $WORDS " in *" $t "*) ;; *) WORDS="$WORDS $t" ;; esac
        fi ;;
      esac ;;
  esac
  prev="$t"
done
# Tags: lowercase, # prefix, de-duplicated.
T2=""
for t in $(printf '%s\n' "$TAGS" | tr 'A-Z' 'a-z'); do
  t="#${t#\#}"; case " $T2 " in *" $t "*) ;; *) T2="$T2 $t" ;; esac
done
TAGS="${T2# }"; WORDS="${WORDS# }"
[ -n "$WORDS$TAGS$FROM" ] || { echo "lookup: nothing to search for. Pass words, --when or --tag (or --show IDs)."; exit 0; }
if [ $ALL = 1 ]; then CAP_M=100000; CAP_O=100000; CAP_N=100000; else CAP_M=15; CAP_O=60; CAP_N=6; fi

TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT
# Every history entry: id  date  file  asked  did  tags  refs  files. Week summaries: file  summary.
for f in $(ls "$MEM"/history/*.md 2>/dev/null | sort); do
  awk -v OFS='\t' -v fn="$(basename "$f")" -v E="$TMP/e.tsv" -v W="$TMP/w.tsv" '
    function clean(s) { gsub(/\t/, " ", s); sub(/ +$/, "", s); return s }
    function val(  v) { v = $0; sub(/^\*\*[A-Za-z]+:\*\* */, "", v); return clean(v) }
    function flush() { if (id != "") print id, date, fn, asked, did, tags, refs, files >> E; id = "" }
    /^### / { flush(); h = substr($0, 5); id = h; sub(/ .*/, "", id); date = h; sub(/^[^·]*· */, "", date)
              asked = did = tags = refs = files = ""; insum = 0; next }
    /^## /  { flush(); insum = ($0 ~ /^## Week summary/); next }
    insum   { if ($0 !~ /^[[:space:]]*$/) sum = sum (sum == "" ? "" : " ") clean($0); next }
    /^\*\*Asked:\*\*/ { asked = val(); next }
    /^\*\*Did:\*\*/   { did = val(); next }
    /^\*\*Tags:\*\*/  { tags = val(); next }
    /^\*\*Refs:\*\*/  { refs = val(); next }
    /^\*\*Files:\*\*/ { files = val(); next }
    END { flush(); if (sum != "") print fn, sum >> W }' "$f"
done
touch "$TMP/e.tsv" "$TMP/w.tsv"

# Score: each query word counts once per field it appears in (Asked 1, Refs/Files 1, Did 2,
# Tags 3). Tags given with #/--tag are filters. Rank: distinct words matched, score, has Did, newest.
export LK_WORDS="$WORDS" LK_TAGS="$TAGS" LK_FROM="$FROM" LK_TO="$TO" LK_SELF="$TURN_ID"
awk -F'\t' -v OFS='\t' '
  function cut(s, n) { if (length(s) > n) { s = substr(s, 1, n - 1); sub(/[\200-\377]+$/, "", s); s = s "…" }; return s }
  BEGIN { nw = split(ENVIRON["LK_WORDS"], w, " "); nt = split(ENVIRON["LK_TAGS"], tg, " ")
          from = ENVIRON["LK_FROM"]; to = ENVIRON["LK_TO"]; self = ENVIRON["LK_SELF"] }
  $1 == self { next }
  {
    d = substr($2, 1, 10)
    tl = " " tolower($6) " "; gsub(/,/, " ", tl)
    for (i = 1; i <= nt; i++) if (index(tl, " " tg[i] " ") == 0) next
    a = tolower($4); di = tolower($5); rf = tolower($7 " " $8)
    hit = sc = 0
    for (i = 1; i <= nw; i++) {
      h = 0
      if (index(a, w[i]))  { sc += 1; h = 1 }
      if (index(di, w[i])) { sc += 2; h = 1 }
      if (index(tl, w[i])) { sc += 3; h = 1 }
      if (index(rf, w[i])) { sc += 1; h = 1 }
      hit += h
    }
    if (from != "" && (d < from || d > to)) { if (hit > 0) print "X"; next }
    line = "- " $1 " | " $2 " | " ($6 == "" ? "-" : $6) " | Asked: " cut($4, 160)
    if ($5 != "") line = line " | Did: " cut($5, 220)
    if ($7 != "") line = line " | Refs: " $7
    if ($8 != "") line = line " | Files: " cut($8, 120)
    if (nw == 0 || hit > 0) print "M", hit, sc, ($5 != ""), $2, line
    else print "O", $2, "- " $1 " | " $2 " | " ($6 == "" ? "-" : $6) " | " cut($4, 100) ($5 == "" ? "" : " | Did: " cut($5, 100))
  }' "$TMP/e.tsv" > "$TMP/scored.tsv"
grep '^M' "$TMP/scored.tsv" | sort -t$'\t' -k2,2nr -k3,3nr -k4,4nr -k5,5r | cut -f6 > "$TMP/m.txt"
grep '^O' "$TMP/scored.tsv" | sort -t$'\t' -k2,2 | cut -f3 > "$TMP/o.txt"
NX=$(grep -c '^X' "$TMP/scored.tsv")

# Notes from REFS.md (generated from the note headers): "- N-id | status | updated | tags | title — summary".
awk -v OFS='\t' '
  BEGIN { nw = split(ENVIRON["LK_WORDS"], w, " "); nt = split(ENVIRON["LK_TAGS"], tg, " ") }
  /^## Notes/ { on = 1; next }  /^## / { on = 0 }
  on && /^### / { dir = $2; next }
  on && /^- N-/ {
    split($0, f, / \| /); id = substr(f[1], 3); tl = " " tolower(f[4]) " "; gsub(/,/, " ", tl)
    for (i = 1; i <= nt; i++) if (index(tl, " " tg[i] " ") == 0) next
    txt = tolower($0); hit = sc = 0
    for (i = 1; i <= nw; i++) if (index(txt, w[i])) { hit++; sc += (index(tl, w[i]) ? 3 : 1) }
    if (hit > 0 || (nw == 0 && nt > 0)) print hit, sc, $0 "  → memory/" dir substr(id, 3) ".md"
  }' "$MEM/REFS.md" 2>/dev/null | sort -t$'\t' -k1,1nr -k2,2nr | cut -f3 > "$TMP/n.txt"

# Week summaries (they describe the entries without a Did).
awk -F'\t' '
  BEGIN { nw = split(ENVIRON["LK_WORDS"], w, " ") }
  { s = tolower($2); hit = 0; for (i = 1; i <= nw; i++) if (index(s, w[i])) hit++
    if (hit > 0) print hit "\t- " $1 " | " $2 }' "$TMP/w.tsv" | sort -t$'\t' -k1,1nr | cut -f2 > "$TMP/wk.txt"

# Report.
desc="words: ${WORDS:-none}"; [ -n "$TAGS" ] && desc="$desc; tags: $TAGS"
[ -n "$FROM" ] && desc="$desc; dates: $FROM .. $TO"
echo "Memory lookup in $MEM ($desc)"
nm=$(wc -l < "$TMP/m.txt" | tr -d ' '); no=$(wc -l < "$TMP/o.txt" | tr -d ' ')
nn=$(wc -l < "$TMP/n.txt" | tr -d ' '); nk=$(wc -l < "$TMP/wk.txt" | tr -d ' ')
section() { # $1 title, $2 file, $3 cap, $4 count
  [ "$4" -gt 0 ] || return 0
  echo; echo "## $1 ($4)"; head -n "$3" "$2"
  [ "$4" -gt "$3" ] && echo "… $(( $4 - $3 )) more (use --all or narrow the query)"
  return 0
}
section "Notes" "$TMP/n.txt" "$CAP_N" "$nn"
section "Matching entries, best first" "$TMP/m.txt" "$CAP_M" "$nm"
[ -n "$FROM" ] && [ -n "$WORDS$TAGS" ] && section "Other entries in $FROM .. $TO, oldest first" "$TMP/o.txt" "$CAP_O" "$no"
section "Matching week summaries" "$TMP/wk.txt" "$CAP_N" "$nk"
[ "$NX" -gt 0 ] && { echo; echo "Also $NX matching entries outside $FROM .. $TO (run again without the time words to see them)."; }
echo
if [ $((nm + nn + nk)) = 0 ] && { [ -z "$FROM" ] || [ "$no" = 0 ]; }; then
  echo "Lookup missed: nothing in notes, refs or history matches ($desc)."
else
  echo "Next: open the notes listed (paths above); full entries with: bash ~/.claude/memory-system/lookup-memory.sh --show <ID>…"
fi
echo "History grep/Read is unlocked for this turn."
exit 0
~~~~

### File 12: `~/.claude/memory-system/init-memory.sh`

~~~~bash
#!/bin/bash
# Enable/disable memory in a project.
#   init-memory.sh [project-dir]             enable (default: current directory); re-enables a disabled project
#   init-memory.sh --disable [project-dir]   opt out, keep notes (renames memory/INDEX.md to INDEX.md.off)
#   init-memory.sh --remove [project-dir]    opt out and delete the memory/ folder (asks for confirmation)
set -e
MODE=enable
case "$1" in
  --disable) MODE=disable; shift ;;
  --remove) MODE=remove; shift ;;
esac
ROOT="$(cd "${1:-$PWD}" && pwd)"
MS="$HOME/.claude/memory-system"
TSV="$HOME/.claude/memory-global/projects.tsv"

unregister() {
  [ -f "$TSV" ] || return 0
  awk -F'\t' -v r="$ROOT" '$2 != r' "$TSV" > "$TSV.tmp" && mv "$TSV.tmp" "$TSV"
}

case "$MODE" in
  disable)
    [ -f "$ROOT/memory/INDEX.md" ] || { echo "not enabled: $ROOT"; exit 0; }
    mv "$ROOT/memory/INDEX.md" "$ROOT/memory/INDEX.md.off"
    unregister
    echo "disabled: $ROOT (notes kept; run init-memory.sh to re-enable)"
    exit 0 ;;
  remove)
    [ -d "$ROOT/memory" ] || { echo "no memory folder: $ROOT"; exit 0; }
    read -r -p "Delete $ROOT/memory and all its notes/history? [y/N] " ans
    [ "$ans" = y ] || [ "$ans" = Y ] || { echo "aborted"; exit 1; }
    rm -rf "$ROOT/memory"
    unregister
    echo "removed: $ROOT/memory"
    exit 0 ;;
esac

if [ -f "$ROOT/memory/INDEX.md" ]; then
  echo "already enabled: $ROOT/memory"
else
  if [ -f "$ROOT/memory/INDEX.md.off" ]; then
    mv "$ROOT/memory/INDEX.md.off" "$ROOT/memory/INDEX.md"
  else
    mkdir -p "$ROOT/memory/history"
    cp "$MS/templates/INDEX.md" "$MS/templates/TAGS.md" "$ROOT/memory/"
  fi
  mkdir -p "$HOME/.claude/memory-global"; touch "$TSV"
  cut -f2 "$TSV" | grep -qxF "$ROOT" || printf '%s\t%s\n' "$(basename "$ROOT")" "$ROOT" >> "$TSV"
  echo "enabled: $ROOT/memory (restart or /clear to see the status line)"
fi

# Run on every enable, also for already-enabled projects. Never re-inits and never commits:
# commits are made only by the Stop hook (check-memory.sh).
if [ ! -d "$ROOT/memory/.git" ]; then
  git -C "$ROOT/memory" init -q
  echo "git initialized: $ROOT/memory"
fi
[ -f "$ROOT/memory/.gitignore" ] || printf '.DS_Store\n' > "$ROOT/memory/.gitignore"
GI="$ROOT/.gitignore"
if ! grep -qxE '/?memory/?' "$GI" 2>/dev/null; then
  [ -s "$GI" ] && [ -n "$(tail -c1 "$GI")" ] && echo >> "$GI"
  echo "/memory/" >> "$GI"
  echo "added /memory/ to $GI"
fi
~~~~

### File 13: `~/.claude/memory-system/sync-memory.sh`

~~~~bash
#!/bin/bash
# Manual memory sync between machines (run via /memory-sync or directly).
# Copies each registered project's memory/ (no .git), ~/.claude/memory-system/ (no state/) and
# ~/.claude/CLAUDE.md into a sync git repo, then: commit -> pull --rebase -> copy back -> push.
# Fixed order so a pull never overwrites unsynced local edits; on conflict it stops before copying back.
# The repo and its remote are set up by the user; this script never changes remote config.
# Never synced (per machine): projects.tsv, GLOBAL_INDEX.md, state/, settings.json, memory/.git,
# and the generated lookup files memory/REFS.md and memory/refs/ (rebuilt on each machine).
set -u
. "$(dirname "$0")/hooks/lib.sh"
SYNC="${MEMORY_SYNC_DIR:-$HOME/claude-memory-sync}"
REG="$GLOBAL/projects.tsv"
# Projects this machine has synced before (local, not committed). A project in the repo but not
# listed here is new to this machine: distribute it first, never collect its fresh template over the repo.
KNOWN="$SYNC/.git/memory-sync-known"
# Files that conflicted on the last run. After the user resolves them in the sync repo, the next
# collect keeps the resolved versions instead of copying this machine's old copies over them.
CONFLICT="$SYNC/.git/memory-sync-conflict"
# The sync repo commit at the conflicted run (this machine's state then); see the collect step.
CONFLICT_BASE="$SYNC/.git/memory-sync-conflict-base"

die() { echo "memory-sync: $*" >&2; exit 1; }
# Copy directory SRC into DST like `rsync -a --delete`, leaving the top-level names given after
# SRC DST untouched on both sides (not copied, and kept in DST). .DS_Store is never copied.
# --no-delete: only add/overwrite. Uses rsync when installed; otherwise tar (Git Bash on Windows
# has no rsync). MEMORY_SYNC_NO_RSYNC=1 forces the tar path.
mirror() {
  local del=1 src dst x n keep
  [ "$1" = --no-delete ] && { del=0; shift; }
  src="${1%/}"; dst="${2%/}"; shift 2
  if [ -z "${MEMORY_SYNC_NO_RSYNC:-}" ] && command -v rsync >/dev/null 2>&1; then
    local a=(-a --exclude .DS_Store)
    [ $del = 1 ] && a+=(--delete)
    for x in "$@"; do a+=(--exclude "/$x"); done
    rsync "${a[@]}" "$src/" "$dst/"
    return
  fi
  mkdir -p "$dst" || return 1
  local names=()
  for n in "$src"/* "$src"/.[!.]* "$src"/..?*; do
    [ -e "$n" ] || continue
    n=$(basename "$n"); keep=0
    for x in "$@" .DS_Store; do [ "$n" = "$x" ] && keep=1; done
    [ $keep = 1 ] || names+=("$n")
  done
  if [ $del = 1 ]; then
    for n in "$dst"/* "$dst"/.[!.]* "$dst"/..?*; do
      [ -e "$n" ] || continue
      keep=0
      for x in "$@"; do [ "$(basename "$n")" = "$x" ] && keep=1; done
      [ $keep = 1 ] || rm -rf "$n"
    done
  fi
  [ ${#names[@]} -gt 0 ] || return 0
  (cd "$src" && tar -cf - --exclude=.DS_Store -- "${names[@]}") | (cd "$dst" && tar -xf -)
}
g() { git -C "$SYNC" "$@"; }

[ -d "$SYNC/.git" ] || die "no sync repo at $SYNC. Set it up: mkdir -p $SYNC && git -C $SYNC init, then add your remote."
[ -d "$SYNC/.git/rebase-merge" ] || [ -d "$SYNC/.git/rebase-apply" ] || [ -f "$SYNC/.git/MERGE_HEAD" ] &&
  die "a pull is still being resolved in $SYNC. Finish it (git add, git rebase --continue), then run again."
[ -f "$REG" ] || die "no projects registered ($REG missing)"
DUP=$(cut -f1 "$REG" | sort | uniq -d)
[ -z "$DUP" ] || die "two registered projects share a name: $DUP. Rename one folder first."
touch "$KNOWN"
known() { grep -qxF "$1" "$KNOWN"; }
mark() { known "$1" || echo "$1" >> "$KNOWN"; }
mkdir -p "$SYNC/projects"
# History files are append-only: keep both machines' lines instead of conflicting. Nothing else
# uses union (INDEX.md is edited in place, so it must conflict instead). Rewritten if it differs.
ATTR='projects/*/history/*.md merge=union'
[ "$(cat "$SYNC/.gitattributes" 2>/dev/null)" = "$ATTR" ] || printf '%s\n' "$ATTR" > "$SYNC/.gitattributes"

# 1. Collect.
while IFS=$'\t' read -r name root; do
  [ -n "$name" ] && [ -f "$root/memory/INDEX.md" ] || continue
  if [ -d "$SYNC/projects/$name" ] && ! known "$name"; then
    continue  # new to this machine: the repo copy wins, filled in by distribute below
  fi
  mirror "$root/memory" "$SYNC/projects/$name" .git REFS.md refs || die "collect failed: $name"
  mark "$name"
done < "$REG"
if [ ! -d "$SYNC/system" ] || known "@system"; then
  mirror "$MS" "$SYNC/system" state .git || die "collect failed: system"
  [ -f "$HOME/.claude/CLAUDE.md" ] && cp "$HOME/.claude/CLAUDE.md" "$SYNC/CLAUDE.md"
  mark "@system"
fi

# After a conflicted run the user pulled and resolved in the sync repo, so HEAD now holds changes
# this machine never received (the resolutions, and the other machine's lines in files that merged
# cleanly). For each file that changed between the conflicted run's commit and HEAD: if this
# machine's copy is still what it was at the conflicted run, keep HEAD's version; if it was edited
# since, keep the local edit. Without this, the collect above would undo the other machine's work.
if [ -s "$CONFLICT_BASE" ] && BASE=$(cat "$CONFLICT_BASE") && g cat-file -e "$BASE^{commit}" 2>/dev/null; then
  g add -A
  g diff --name-only "$BASE" HEAD | while IFS= read -r f; do
    g diff --cached --quiet "$BASE" -- "$f" || continue
    if g cat-file -e "HEAD:$f" 2>/dev/null; then g checkout -q HEAD -- "$f"; else g rm -q --cached --ignore-unmatch -- "$f"; rm -f "$SYNC/$f"; fi
  done
elif [ -s "$CONFLICT" ]; then
  # Older state without a recorded base: keep the resolved versions of the conflicted files.
  while IFS= read -r f; do
    if g cat-file -e "HEAD:$f" 2>/dev/null; then g checkout -q HEAD -- "$f"; else rm -f "$SYNC/$f"; fi
  done < "$CONFLICT"
fi

# 2. Commit.
g add -A
if ! g diff --cached --quiet; then
  g commit -q -m "memory sync $(hostname 2>/dev/null | cut -d. -f1) $(date '+%Y-%m-%d %H:%M')" || die "commit failed"
fi

# 3. Pull.
PULLED=0; PUSH=no; REMOTE=yes
if [ -z "$(g remote)" ]; then
  REMOTE=no
else
  g fetch -q || die "fetch failed (network or auth?). Nothing was copied back."
  if ! g rev-parse -q --verify '@{u}' >/dev/null 2>&1; then
    BR=$(g branch --show-current)
    R=$(g remote | head -1)
    g rev-parse -q --verify "refs/remotes/$R/$BR" >/dev/null && g branch -q --set-upstream-to="$R/$BR"
  fi
  if g rev-parse -q --verify '@{u}' >/dev/null 2>&1; then
    PULLED=$(g rev-list --count 'HEAD..@{u}')
    if [ "$PULLED" -gt 0 ]; then
      if ! g rebase -q '@{u}' >/dev/null 2>&1; then
        echo "memory-sync: conflict while pulling. Nothing was copied back or pushed. Conflicting files:"
        g diff --name-only --diff-filter=U > "$CONFLICT"
        sed 's/^/  /' "$CONFLICT"
        g rebase --abort
        g rev-parse HEAD > "$CONFLICT_BASE"
        echo "Resolve: cd $SYNC && git pull --rebase, fix the files, git add them, git rebase --continue. Then run /memory-sync again."
        exit 1
      fi
    fi
    [ "$(g rev-list --count '@{u}..HEAD')" -gt 0 ] && PUSH=yes
  else
    PUSH=first
  fi
fi

# 4. Distribute.
echo "memory-sync:"
LOCAL_NAMES=$(cut -f1 "$REG")
while IFS=$'\t' read -r name root; do
  [ -n "$name" ] || continue
  if [ ! -d "$SYNC/projects/$name" ] || [ ! -d "$root/memory" ]; then
    echo "  $name: not in sync repo"; continue
  fi
  if diff -rq -x .git -x .DS_Store -x REFS.md -x refs "$SYNC/projects/$name" "$root/memory" >/dev/null 2>&1; then
    echo "  $name: no incoming changes"
  else
    mirror "$SYNC/projects/$name" "$root/memory" .git REFS.md refs || die "copy back failed: $name"
    echo "  $name: received changes"
  fi
  mark "$name"
done < "$REG"
for d in "$SYNC/projects"/*/; do
  [ -d "$d" ] || continue
  n=$(basename "$d")
  printf '%s\n' "$LOCAL_NAMES" | grep -qxF "$n" || echo "  $n: not set up here"
done
if [ -d "$SYNC/system" ]; then
  mirror --no-delete "$SYNC/system" "$MS" state .git || die "copy back failed: system"
  [ -f "$SYNC/CLAUDE.md" ] && cp "$SYNC/CLAUDE.md" "$HOME/.claude/CLAUDE.md"
  mark "@system"
fi

# 5. Push.
case "$REMOTE/$PUSH" in
  no/*) echo "  remote: none set, skipped pull/push" ;;
  yes/first) g push -q -u "$(g remote | head -1)" HEAD >/dev/null 2>&1 && echo "  remote: first push done" || die "push failed" ;;
  yes/yes) g push -q >/dev/null 2>&1 && echo "  remote: pulled $PULLED, pushed" || die "push failed (pulled $PULLED, copied back)" ;;
  yes/no) echo "  remote: pulled $PULLED, nothing to push" ;;
esac

rm -f "$CONFLICT" "$CONFLICT_BASE"

# 6. Rebuild this machine's global index.
bash "$MS/hooks/build-global-index.sh" 2>/dev/null
exit 0
~~~~

### File 14: `~/.claude/memory-system/install.sh`

~~~~bash
#!/bin/bash
# Install / update / uninstall the Claude memory system (files already in ~/.claude/memory-system/).
#   install.sh --email you@example.com [--no-sync] [--sync-dir DIR]
#                                                     wire hooks, rules and the /memory-sync skill;
#                                                     DIR (default ~/claude-memory-sync) is stored per
#                                                     machine in settings.json "env"
#   install.sh --uninstall                            remove hooks, rules and skill (keeps the scripts
#                                                     and every project's memory/ folder)
# Safe to re-run: settings.json entries are only added when missing, the rules block in
# ~/.claude/CLAUDE.md is replaced in place, and backups are written before each change.
set -u
MS="$HOME/.claude/memory-system"
SET="$HOME/.claude/settings.json"
CMD="$HOME/.claude/CLAUDE.md"
SKILL="$HOME/.claude/skills/memory-sync"
START='<!-- claude-memory:start -->'
END='<!-- claude-memory:end -->'
STAMP=$(date +%Y%m%d-%H%M%S)

die() { echo "install: $*" >&2; exit 1; }
say() { echo "install: $*"; }

EMAIL=""; SYNC=1; MODE=install; SYNC_DIR=""
while [ $# -gt 0 ]; do
  case "$1" in
    --email) EMAIL="${2:-}"; shift ;;
    --email=*) EMAIL="${1#--email=}" ;;
    --no-sync) SYNC=0 ;;
    --sync-dir) SYNC_DIR="${2:-}"; shift ;;
    --sync-dir=*) SYNC_DIR="${1#--sync-dir=}" ;;
    --uninstall) MODE=uninstall ;;
    *) die "unknown option $1" ;;
  esac
  shift
done

for c in bash git jq awk sed find mktemp; do
  command -v "$c" >/dev/null 2>&1 || die "missing required command: $c (see the Requirements section)"
done
[ -f "$MS/hooks/lib.sh" ] || die "scripts not found in $MS; write the files first"

backup() { [ -f "$1" ] && cp "$1" "$1.bak-$STAMP" && say "backup: $1.bak-$STAMP"; return 0; }

# Remove the rules block (between the markers) from CLAUDE.md, or replace it with file $1.
put_block() {
  local blk="${1:-}" tmp
  touch "$CMD"
  tmp=$(mktemp)
  awk -v s="$START" -v e="$END" -v f="$blk" '
    BEGIN { if (f != "") while ((getline l < f) > 0) b = b l "\n" }
    $0 == s { skip = 1; if (f != "" && !done) { printf "%s", b; done = 1 }; next }
    $0 == e { skip = 0; next }
    !skip { print }
    END { if (f != "" && !done) { if (NR > 0) print ""; printf "%s", b } }' "$CMD" > "$tmp" && cat "$tmp" > "$CMD"
  rm -f "$tmp"
}

# Hook entries this system needs (merged into settings.json, never replacing other hooks).
FRAG=$(cat <<'JSON'
{
  "hooks": {
    "UserPromptSubmit": [
      { "hooks": [ { "type": "command", "command": "bash ~/.claude/memory-system/hooks/log-prompt.sh" } ] }
    ],
    "PreToolUse": [
      { "matcher": "Write|Edit|NotebookEdit", "hooks": [ { "type": "command", "command": "bash ~/.claude/memory-system/hooks/guard-memory.sh" } ] },
      { "matcher": "Read|Grep|Glob|Bash", "hooks": [ { "type": "command", "command": "bash ~/.claude/memory-system/hooks/guard-memory-read.sh" } ] }
    ],
    "PostToolUse": [
      { "matcher": "Write|Edit|NotebookEdit", "hooks": [ { "type": "command", "command": "bash ~/.claude/memory-system/hooks/log-edit.sh" } ] },
      { "matcher": "Read|Grep|Glob|Bash", "hooks": [ { "type": "command", "command": "bash ~/.claude/memory-system/hooks/guard-memory-read.sh --approved" } ] }
    ],
    "Stop": [
      { "hooks": [ { "type": "command", "command": "bash ~/.claude/memory-system/hooks/check-memory.sh", "timeout": 30 } ] }
    ],
    "SessionStart": [
      { "matcher": "startup|clear", "hooks": [ { "type": "command", "command": "bash ~/.claude/memory-system/hooks/session-status.sh" } ] }
    ]
  },
  "permissions": { "allow": [ "Bash(bash ~/.claude/memory-system/lookup-memory.sh:*)" ] }
}
JSON
)

if [ "$MODE" = uninstall ]; then
  if [ -f "$SET" ]; then
    backup "$SET"
    tmp=$(mktemp)
    jq '
      if .hooks then
        .hooks |= (with_entries(.value |= map(select([.hooks[]?.command // ""] | any(test("memory-system/")) | not)))
                   | with_entries(select(.value | length > 0)))
      else . end
      | if .permissions.allow then .permissions.allow |= map(select(test("memory-system/") | not)) else . end
      | if .env.MEMORY_SYNC_DIR then del(.env.MEMORY_SYNC_DIR) | (if .env == {} then del(.env) else . end) else . end' \
      "$SET" > "$tmp" && jq -e . "$tmp" >/dev/null && cat "$tmp" > "$SET" || die "could not update $SET"
    rm -f "$tmp"
    say "settings.json: memory hooks and permission removed"
  fi
  if [ -f "$CMD" ] && grep -qxF "$START" "$CMD"; then
    backup "$CMD"; put_block ""; say "CLAUDE.md: rules block removed"
  fi
  rm -rf "$SKILL" && say "skill /memory-sync removed"
  say "done. Scripts stay in $MS and every project's memory/ folder is kept."
  say "To delete the scripts too: rm -rf ~/.claude/memory-system ~/.claude/memory-global"
  exit 0
fi

case "$SYNC_DIR" in "~"*) SYNC_DIR="$HOME${SYNC_DIR#\~}" ;; esac
[ -n "$EMAIL" ] || die "pass --email <your git author email> (used for the 'only my own work' rule)"
case "$EMAIL" in *'|'*|*'\'*|*'&'*) die "unexpected characters in email" ;; esac

# 1. Files: executable bits, no stale state.
chmod +x "$MS"/*.sh "$MS"/hooks/*.sh
mkdir -p "$MS/state" "$HOME/.claude/memory-global"
say "scripts ready in $MS"

# 2. settings.json: add missing hook entries and the lookup permission, keep everything else.
mkdir -p "$HOME/.claude"
[ -s "$SET" ] || echo '{}' > "$SET"
jq -e . "$SET" >/dev/null 2>&1 || die "$SET is not valid JSON; fix it first"
backup "$SET"
tmp=$(mktemp)
printf '%s' "$FRAG" > "$tmp.frag"
jq --slurpfile f "$tmp.frag" '
  reduce ($f[0].hooks | to_entries[]) as $e (.;
    .hooks[$e.key] = ((.hooks[$e.key] // []) as $cur
      | $cur + [ $e.value[] | select(.hooks[0].command as $c | ([$cur[]?.hooks[]?.command] | index($c)) == null) ]))
  | .permissions.allow = ((.permissions.allow // []) as $a
      | $a + [ $f[0].permissions.allow[] | select(. as $x | ($a | index($x)) == null) ])
  | if $dir != "" then .env.MEMORY_SYNC_DIR = $dir else . end' \
  --arg dir "$SYNC_DIR" "$SET" > "$tmp" && jq -e . "$tmp" >/dev/null && cat "$tmp" > "$SET" || die "could not update $SET"
rm -f "$tmp" "$tmp.frag"
say "settings.json: hooks and lookup permission merged"
[ -n "$SYNC_DIR" ] && say "settings.json: env MEMORY_SYNC_DIR=$SYNC_DIR"

# 3. Rules in ~/.claude/CLAUDE.md, between markers (replaced in place on re-install).
blk=$(mktemp)
{ echo "$START"; sed "s|{{GIT_EMAIL}}|$EMAIL|g" "$MS/templates/CLAUDE-section.md"; echo "$END"; } > "$blk"
[ -f "$CMD" ] && backup "$CMD"
put_block "$blk"
rm -f "$blk"
say "CLAUDE.md: rules block written"

# 4. /memory-sync skill (optional).
if [ $SYNC = 1 ]; then
  mkdir -p "$SKILL" && cp "$MS/templates/memory-sync-SKILL.md" "$SKILL/SKILL.md"
  say "skill /memory-sync installed"
else
  rm -rf "$SKILL"
  say "skill /memory-sync not installed (--no-sync)"
fi
command -v rsync >/dev/null 2>&1 || say "note: rsync not found; sync will use its tar fallback"

say "done. Run: bash ~/.claude/memory-system/selftest.sh"
say "Then enable a project: bash ~/.claude/memory-system/init-memory.sh /path/to/project"
say "Restart Claude Code (or open /hooks once) so the hooks load."
exit 0
~~~~

### File 15: `~/.claude/memory-system/selftest.sh`

~~~~bash
#!/bin/bash
# Self-test of the installed memory system. Runs in a throwaway sandbox (fake HOME, two fake
# projects) and never touches your real projects, settings, registry or state.
#   bash ~/.claude/memory-system/selftest.sh
# Prints PASS/FAIL per check; exit status is the number of failures.
set -u
SRC="$HOME/.claude/memory-system"
[ -f "$SRC/hooks/lib.sh" ] || { echo "selftest: $SRC not installed"; exit 1; }
for c in bash git jq awk sed; do command -v "$c" >/dev/null 2>&1 || { echo "selftest: missing $c"; exit 1; }; done

# No EXIT trap: older bash (macOS 3.2) can run it when a $(...) subshell exits.
T=$(cd "$(mktemp -d)" && pwd -P)
trap 'rm -rf "$T"; exit 130' INT TERM
export HOME="$T/home"
mkdir -p "$HOME/.claude"
cp -R "$SRC" "$HOME/.claude/memory-system"
rm -rf "$HOME/.claude/memory-system/state"
MS="$HOME/.claude/memory-system"; H="$MS/hooks"
P="$T/proj"; Q="$T/other"; mkdir -p "$P" "$Q"
export CLAUDE_PROJECT_DIR="$P"
export GIT_AUTHOR_NAME=selftest GIT_AUTHOR_EMAIL=selftest@example.com GIT_COMMITTER_NAME=selftest GIT_COMMITTER_EMAIL=selftest@example.com

FAILS=0
check() { if eval "$2" >/dev/null 2>&1; then echo "PASS  $1"; else echo "FAIL  $1"; FAILS=$((FAILS + 1)); fi; }
hook() { local h="$1"; shift; printf '%s' "$1" | bash "$H/$h" "${@:2}"; }
grepq() { local d="$1"; shift; jq -n --arg t Grep --arg p "$d" --arg c "$P" '{session_id:"s",tool_name:$t,tool_input:{pattern:"x",path:$p},cwd:$c}' | bash "$H/guard-memory-read.sh" "${@:2}"; }

bash "$MS/init-memory.sh" "$P" >/dev/null 2>&1
check "enable a project (init-memory.sh)" '[ -f "$P/memory/INDEX.md" ] && [ -f "$P/memory/TAGS.md" ] && [ -d "$P/memory/.git" ]'

hook log-prompt.sh '{"prompt":"add a login screen"}' >/dev/null
HF=$(ls "$P"/memory/history/*.md 2>/dev/null | head -1)
check "prompt logged to this week's history file" 'grep -q "^\*\*Asked:\*\* add a login screen" "$HF"'
ID=$(grep -o '^### H-[0-9W]*-[0-9]*' "$HF" | tail -1 | cut -c5-)

touch "$P/app.txt"
hook log-edit.sh "{\"tool_input\":{\"file_path\":\"$P/app.txt\"}}"
hook check-memory.sh '{}' >/dev/null 2>&1; rc=$?
check "Stop blocks: code edited without a note or Did" '[ $rc = 2 ]'

mkdir -p "$P/memory/project"
cat > "$P/memory/project/feature-login.md" <<'EOF'
# Login screen
ID: N-feature-login
Tags: #feature
Summary: Login screen with email and password
Status: done    Last updated: 2000-01-01
Last change: Created the note.
## Goal / Context
Users sign in before using the app.
EOF
printf '**Did:** Added the login screen.\n**Tags:** #feature\n**Refs:** N-feature-login\n' >> "$HF"
OUT=$(hook check-memory.sh '{}' 2>&1); rc=$?
check "Stop passes once note and Did exist" '[ $rc = 0 ]'
check "memory summary shown to the user" 'printf %s "$OUT" | grep -q "memory updated"'
check "REFS.md generated with the note" 'grep -q "N-feature-login" "$P/memory/REFS.md"'
check "monthly refs file lists the entry" 'grep -q "$ID" "$P"/memory/refs/*.md'
check "memory/ committed to its own git repo" '[ -n "$(git -C "$P/memory" log --oneline 2>/dev/null)" ]'

hook log-prompt.sh '{"prompt":"when did we add login?"}' >/dev/null
grepq "$P/memory/history" >/dev/null 2>&1; rc=$?
check "gate blocks grepping history before the lookup" '[ $rc = 2 ]'
LK=$(cd "$P" && bash "$MS/lookup-memory.sh" 2>&1)
check "lookup finds the entry and the note" 'printf %s "$LK" | grep -q "$ID" && printf %s "$LK" | grep -q "N-feature-login"'
grepq "$P/memory/history" >/dev/null 2>&1; rc=$?
check "gate unlocked after the lookup" '[ $rc = 0 ]'
SH=$(cd "$P" && bash "$MS/lookup-memory.sh" --show "$ID" 2>&1)
check "lookup --show prints the full entry" 'printf %s "$SH" | grep -q "Added the login screen"'

bash "$MS/init-memory.sh" "$Q" >/dev/null 2>&1
hook guard-memory.sh "{\"tool_input\":{\"file_path\":\"$Q/memory/x.md\"}}" >/dev/null 2>&1; rc=$?
check "writing another project's memory is blocked" '[ $rc = 2 ]'
R=$(jq -n --arg f "$Q/memory/INDEX.md" '{session_id:"s2",tool_name:"Read",tool_input:{file_path:$f},cwd:"/"}' | bash "$H/guard-memory-read.sh" 2>/dev/null)
check "reading another project's memory asks first" 'printf %s "$R" | grep -q "\"ask\""'
bash "$H/build-global-index.sh"
check "global index lists notes across projects" 'grep -q "N-feature-login" "$HOME/.claude/memory-global/GLOBAL_INDEX.md"'

hook log-prompt.sh '{"prompt":"re-arm the gate"}' >/dev/null
printf 'exit 2\n' > "$H/guard-history.sh"
grepq "$P/memory/history" >/dev/null 2>&1; rc=$?
check "a broken gate never blocks tools" '[ $rc = 0 ]'

rm -rf "$T"
echo
[ $FAILS = 0 ] && echo "selftest: all checks passed" || echo "selftest: $FAILS check(s) failed"
exit $FAILS
~~~~

### File 16: `~/.claude/memory-system/templates/INDEX.md`

~~~~markdown
# Memory Index

This project's notebook: what the user built, fixed and improved, problems hit, permissions/config needed, and what is left to do. Read this first, then the notes for the area you are about to change. Rules live in `~/.claude/CLAUDE.md`.

## Project locations / orientation
(add where things live; keep short)

## How-to
(repeatable procedures, in prose; the notes themselves are listed in the generated `REFS.md`)

The list of notes is generated: see `REFS.md` (never edit it). Each note's `Summary:` line is what appears there.
~~~~

### File 17: `~/.claude/memory-system/templates/TAGS.md`

~~~~markdown
# Tags
Controlled vocabulary. Reuse; add a new tag (with a one-line meaning) only when none fits.

- #feature — new functionality
- #bugfix — a defect fixed
- #improvement — refactor or enhancement
- #config — permissions, env, dependencies, setup
- #decision — a choice and its reason
- #question — a lookup or answer worth finding later
~~~~

### File 18: `~/.claude/memory-system/templates/CLAUDE-section.md`

~~~~markdown
## Project Memory System

A project is memory-enabled when `memory/INDEX.md` exists at its root (enable with `~/.claude/memory-system/init-memory.sh`). If it does not exist, skip everything below. The user's own notebook: written automatically, organized, survives `/clear`. Hooks (global, `~/.claude/memory-system/hooks/`) enforce it; they only act in enabled projects.

**Read before changing code:** read `memory/INDEX.md` (orientation) and `memory/REFS.md` (the generated note list), then the notes for the area being touched; use their decisions, gotchas, permissions and TODOs.

**Answer from the current project's memory only.** Use `~/.claude/memory-global/GLOBAL_INDEX.md` (generated, notes only; never edit) or another project's `memory/` only when the user explicitly asks about other projects or all their memory. A hook (`guard-memory-read.sh`) asks the user before the first such read in a session; once approved, the rest of the session is unlocked.

**Only the user's own work** (git author `{{GIT_EMAIL}}`): changes Claude makes for them, changes they say they made. Not teammates' commits or pulled code, except the impact on the user's code in one line. If unclear, ask in one line. Never record secrets, credentials or tokens in `memory/`.

**Write in the same turn** when: code changes for a feature/fix/improvement; a problem is hit and solved; a permission/config/env/dependency is needed; a decision is made (with the reason); the user mentions a TODO or gotcha; a recorded fact turns out wrong; the user gives durable orientation info (where things live, how to run/test). Skip trivial edits.

**Only the current project's memory:** write notes, history and any file under `memory/` only in the project this session started in, never in another project's `memory/`, even when the work concerns it. A PreToolUse hook blocks it. To record something about another project, tell the user to do it from a session started there.

**Store** (`memory/` at the project root, own git repo, auto-committed by the Stop hook, never pushed; never run git in it yourself): `INDEX.md` (orientation, locations and how-tos only; no note list), `TAGS.md` (controlled tags), `REFS.md` and `refs/YYYY-MM.md` (GENERATED from notes and history, never edit), `history/YYYY-Www.md` (one file per week, weeks start on Sunday), and notes in area folders as `<folder>/<type>-<name>.md` (type: feature-, bugfix-, improvement-, change-; lowercase kebab-case; ID is `N-<filename>`).

**Note template:**
```
# <name>
ID: N-<filename without .md>
Tags: #tag #tag
Summary: <one line: what this note is about; shown in REFS.md>
Status: in progress | done | blocked    Last updated: YYYY-MM-DD
Last change: <1-2 plain sentences, max ~35 words: which section changed and what was written>
## Goal / Context
## What changed (files, classes, entry points)
## Decisions & why
## Permissions / config / env needed
## Issues faced & how they were solved
## Gotchas
## TODO / future work
## Related notes
```
Omit empty sections. Keep notes short; prefer one line in an existing note over a new file. Update in place, rewrite stale parts, keep `Summary:` current, rewrite `Last change:` every time (the Stop hook shows it to the user). Don't say "I updated memory" in replies.

**History:** the hook writes each entry's header, `Asked:` and `Files:`; never edit those or invent IDs. IDs name their file: `H-2026W42-0042` is in `history/2026-W42.md`. Add `**Did:**` (1-2 sentences), `**Tags:**` (1-3 from TAGS.md) and optional `**Refs:**` (note IDs) directly below this turn's `**Asked:**` for every turn that did real work. **Required whenever you edited any file this turn** (project, memory notes, or outside the project): the Stop hook blocks until the Did is there. For an answer worth finding later, add a Did tagged `#question`. Entries without a Did are listed in refs only as one line per week; when the prompt hook asks for last week's `## Week summary`, append it first (one paragraph, 2-3 sentences, at the end of that week's file).

**Lookup:** 1) run `bash ~/.claude/memory-system/lookup-memory.sh` first: no args uses this turn's question; or pass words, `--when today|yesterday|this-week|last-week|YYYY-MM-DD|YYYY-MM`, `--tag '#x'`. It searches the notes, every history entry (including those without a Did) and the week summaries, and lists matches plus the other entries in the time window. 2) Open the notes it lists; print full entries with `lookup-memory.sh --show <ID>…`. 3) Only if that is not enough: grep `memory/history/*.md`. A hook (`guard-history.sh`) blocks grepping or reading history until the script has run in the current turn. Answer with ID, date, asked/done, files. If nothing matches, say the lookup missed. Never fall back to GLOBAL_INDEX.md or other projects unless the user explicitly asks.

**Limits:** only edits through Claude's Write/Edit tools are seen by hooks. If hooks seem inactive after a settings change, open `/hooks` once or restart `claude`.
~~~~

### File 19: `~/.claude/memory-system/templates/memory-sync-SKILL.md`

~~~~markdown
---
name: memory-sync
description: Sync Claude memory (all registered projects' memory/, the memory-system scripts, ~/.claude/CLAUDE.md) with your other machines through a git repo. Manual only.
disable-model-invocation: true
---

Run:

```bash
bash ~/.claude/memory-system/sync-memory.sh
```

Report its output to the user as-is (one line per project plus the remote line).

- If it reports a conflict, list the conflicting files and stop. Do not resolve, edit, or retry automatically; the user resolves it in the sync repo (default `~/claude-memory-sync`, or `$MEMORY_SYNC_DIR`) and runs `/memory-sync` again.
- If it says there is no sync repo, show the setup line it printed. Do not create the repo or add a remote yourself.
- Do not read or summarize other projects' notes; only report the script's output.
~~~~


---

## Step 3: Run the installer

Use the Step 1 answers. Add `--no-sync` if sync was declined. Add `--sync-dir` only if the user chose a folder other than the default.

```bash
bash ~/.claude/memory-system/install.sh --email "<email from step 1>"
# variants:
bash ~/.claude/memory-system/install.sh --email "<email>" --no-sync
bash ~/.claude/memory-system/install.sh --email "<email>" --sync-dir "<folder>"
```

The installer:

1. Makes the scripts executable.
2. Backs up `settings.json` and merges in the hook entries and the lookup permission. It adds only missing entries and keeps every existing hook and setting.
3. Writes the rules block into `~/.claude/CLAUDE.md` between `<!-- claude-memory:start -->` and `<!-- claude-memory:end -->`, after a backup. Any other content stays.
4. Installs the `/memory-sync` skill, unless `--no-sync` is given.

It is safe to run again.

---

## Step 4: Self-test

```bash
bash ~/.claude/memory-system/selftest.sh
```

It runs in a throwaway sandbox (fake home, fake projects) and never touches real projects or settings. Every line must say `PASS`. If any line says `FAIL`, show the full output to the user and stop.

---

## Step 5: Enable a project

Ask which project folder to enable. Usually that is the current working directory, if it is a project. Then run:

```bash
bash ~/.claude/memory-system/init-memory.sh "<project folder>"
```

This creates `<project>/memory/` (INDEX.md, TAGS.md, history/, its own git repo) and adds `/memory/` to the project's `.gitignore`, so notes never land in the code repo.

Then tell the user:

- **Restart Claude Code** (or open `/hooks` once) so the new hooks load.
- On the next session start, a status line says `memory: enabled — N notes, last entry …`.
- To enable more projects, run the same command once per project.
- To turn memory off for a project:
  - `init-memory.sh --disable <dir>` keeps the notes;
  - `init-memory.sh --remove <dir>` deletes them, and asks first.

---

## Step 6 (optional): Set up sync between machines

Only if sync was chosen. **The user creates the remote. Claude never creates repos or adds remotes for them.**

Tell the user:

1. Create an **empty, private** git repository on their git host (GitHub, GitLab, a server). It is private because it contains their notes and every prompt they typed.
2. On **each** machine, in a terminal, using the sync folder from Step 1:
   ```bash
   mkdir -p ~/claude-memory-sync
   git -C ~/claude-memory-sync init
   git -C ~/claude-memory-sync remote add origin <private-repo-url>
   ```
3. Enable the same projects on each machine with `init-memory.sh`. Projects are matched by **folder name**, so give each project the same folder name everywhere.
4. Run `/memory-sync` in Claude Code on the first machine, which does the first push. Then run it on the others; they receive.
5. From then on, run `/memory-sync` whenever switching machines.

How sync behaves:

- **What is synced:** each project's `memory/` (without its `.git`), the scripts in `~/.claude/memory-system/` and `~/.claude/CLAUDE.md`. The whole CLAUDE.md file is synced, so keep it the same on all machines.
- **What is never synced:** `settings.json`, the project registry, `state/`, the generated `REFS.md` and `refs/` (rebuilt on each machine).
- **Conflicts:** history files merge line by line, so added entries never clash. If any other file changed on both machines, sync stops and copies nothing back. The user resolves it in the sync folder (`git pull --rebase`, fix, `git add`, `git rebase --continue`) and runs `/memory-sync` again.

---

## Updating, uninstalling

- **Update:** run this whole procedure again with the new version of this file. The files are replaced and the settings and rules block are re-merged. Notes are untouched.
- **Uninstall:** `bash ~/.claude/memory-system/install.sh --uninstall`.
  - It removes the hook entries, the permission, the sync folder setting, the rules block and the skill, with backups.
  - It keeps the scripts and every project's `memory/`.
  - To remove the scripts too: `rm -rf ~/.claude/memory-system ~/.claude/memory-global`.
  - Then restart Claude Code.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| No `memory:` status line at session start | Restart Claude Code or open `/hooks` once. Check that the project has `memory/INDEX.md`. |
| `jq: command not found` in hook errors | Install jq (Step 0) and restart Claude Code. |
| Windows: `bash: command not found` | Add Git's `bin` folder to PATH, then restart. |
| The turn is "sent back" asking for a note or Did | This is intended: Claude edited files without updating memory. Claude adds it and continues. |
| "Memory lookup order: run lookup-memory.sh first" | This is intended: history is searched through the lookup script first. Claude runs it, and grep works after that. |
| `memory/` never gets commits | Git needs an identity: `git config --global user.name "..."` and `git config --global user.email "..."`. |
