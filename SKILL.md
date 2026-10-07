---
name: recall
description: "Tree-structured, feature-segmented project memory. Use when: (1) the user enters a project containing a `.recall/` directory and asks an open-ended status question like 'what was I working on', 'continue where we left off', 'pick up where I left off', 'where did we leave off'; (2) the user explicitly invokes `/recall`, `/recall status`, `/recall load <feature>`, `/recall dump`, `/recall tree`, `/recall new`, `/recall search <query>`, or `/recall doctor`; (3) the user says 'save context', 'context dump', 'remember this session'; (4) a conversation is wrapping up ('ok I'm done', 'wrap up', 'that's it for today') in a project with `.recall/` — nudge to dump. Stores per-project, per-feature memory in `.recall/` so future sessions can resume with minimal context cost."
---

# recall

Persistent, tree-structured memory for a project. Lets future Claude sessions resume work without bloating the context window — only the relevant feature branches load on demand.

## Mental model: three tiers

| Tier | Files | When loaded |
|---|---|---|
| **L1 — always hot** | `.recall/tree.md`, `.recall/recent.md`, `.recall/project.md` | Every resume. Tiny (~1–4 KB). The index + project facts. |
| **L2 — feature summary** | `.recall/features/<f>/overview.md`, `decisions.md` | Only when user targets feature `<f>`. |
| **L3 — session detail** | `.recall/features/<f>/sessions/YYYY-MM-DD-<slug>.md` | Only on explicit deep-dive. Heavy. |

**Rule of thumb:** never load L3 unless you have a clear reason. Most resumes only need L1.

## Directory layout

```
.recall/
├── config.md               # sharing mode (personal/shared) + settings
├── tree.md                 # feature index — ALWAYS load on resume
├── recent.md               # last 3 sessions across all features
├── project.md              # project-level facts: stack, conventions, entry points
└── features/
    └── <feature>/
        ├── overview.md     # current state, open threads, key files
        ├── decisions.md    # why things are the way they are
        ├── archive.md      # consolidated summary of old sessions (created on demand)
        └── sessions/
            └── YYYY-MM-DD-<slug>.md
```

Templates for every file are in `templates/` next to this SKILL.md.

## Commands

### `/recall` or `/recall status`
1. Read `.recall/tree.md`, `.recall/recent.md`, and `.recall/project.md` (L1 only).
2. **Staleness check** (git repos only). Find the oldest `last:` date among features marked `[active]`, then run once:
   ```
   git log --since=<oldest-active-last> --name-only --pretty=format:
   ```
   Match the changed file paths against each active feature's `paths:` globs. A feature whose paths saw commits after its own `last:` date is stale — code moved without a dump (work outside Claude, or a session that never dumped). Skip `[stable]`/`[archived]` features and skip the check entirely if the repo has no commits or `.recall/` is gitignored-personal in a repo with heavy unrelated churn.
3. Summarize: "You were last working on `<feature>/<slug>` — `<one-line state>`. Continue, or something else?" Append one line per stale feature: "⚠ `<feature>`: N commits since last dump (YYYY-MM-DD) — overview may be stale."
4. Do NOT load any L2 or L3 yet. Staleness warnings change what you *trust*, not what you *load* — if the user continues on a stale feature, load its L2 as usual but weight `git log`/`git diff` over `overview.md` where they disagree.

### `/recall load <feature>` (or implicit, when user targets a feature)
1. Read `.recall/features/<feature>/overview.md` and `decisions.md`.
2. Only read the latest session file under `sessions/` if `overview.md`'s "Recent sessions" list marks it `[deep]` (see template — the flag means "this session file holds detail not captured in the overview") AND the user is going deep on that thread.
3. If multiple features are mentioned, load each one's overview only — no session detail until asked.

### `/recall dump [feature]`
This is the most important flow. Run at end of a working session.

**Step 0 — first run check.** If `.recall/` doesn't exist, prompt: "No `.recall/` here yet. Create personal (gitignored) or shared (committed) memory?" Then scaffold `.recall/` with templates and write the choice to `config.md`. If gitignored, append `.recall/` to `.gitignore` (create the file if needed).

**Step 1 — collect file evidence.** From your own tool-call history this session, list every file path passed to Read/Edit/Write/Bash. Also run `git diff --name-only` if it's a git repo. Deduplicate. In a git repo, also capture the repo anchor in one command — `git rev-parse --abbrev-ref HEAD && git rev-parse --short HEAD && git status --porcelain | wc -l` — it fills the session file's "Repo state" line, tying the memory to a point in git history (future sessions can `git diff <short_sha>..HEAD` to see exactly what changed since the dump).

**Step 2 — route to features.** Read `tree.md`. For each feature, score it by how many touched files match its `paths:` globs.
- One feature dominates (≥70% of touched files): assign there.
- Two features close: split the dump — write one session file per feature, each containing only the relevant slice. Cross-reference them with a shared session ID at the top.
- Nothing matches: ask the user "this touched `<paths>` — assign to existing feature (`auth`/`billing`/...) or create new?"

**Step 3 — confirm.** Show the assignment ("Saving to `auth/` (matched 8/10 touched files). OK?") before writing. Cheap to correct.

**Step 4 — write.** For each affected feature:
1. Create `features/<f>/sessions/YYYY-MM-DD-<slug>.md` from `templates/session.md`. Use today's date. Slug should be 2–4 words describing the work. If a same-day file with the same slug exists, append `-2`, `-3`, etc.
2. **Edit** (not overwrite) `features/<f>/overview.md`:
   - Replace the "Current state" section with the new state.
   - Replace "Open threads" with the updated todo list.
   - Append the new session to "Recent sessions" (most recent first, keep last ~5). Tag the entry `[deep]` if the session file holds detail worth re-reading that the overview can't capture (tricky debugging chain, half-finished refactor mid-flight); leave untagged otherwise.
3. Append each entry from the session's "Decisions made" section to `features/<f>/decisions.md` (date-prefixed, one line of "why" each). Skip if no decisions were made.
4. Update `tree.md`: bump `last:` for this feature, update its one-line summary if state changed.
5. Update `recent.md`: prepend the new entry, drop oldest if > 3.
6. If project-level facts changed (stack, conventions, entry points, build/test commands), update the relevant section of `project.md`. Most dumps skip this.

**Step 5 — archive check.** If a feature now has > 5 session files, ask the user: "auth has 7 sessions. Archive the 4 oldest into `archive.md`?" If yes, summarize them as bullets in `archive.md`, then delete the old session files.

### `/recall tree`
Read and display `.recall/tree.md` verbatim.

### `/recall new <feature> [parent]`
1. Create `features/<feature>/` with `overview.md` and `decisions.md` from templates.
2. Propose initial `paths:` globs based on files touched in the current conversation. Ask user to confirm.
3. Add the new feature to `tree.md` (under `parent` if given, else top-level).

### `/recall search <query>`
Full-text search across ALL of `.recall/` — sessions and archives included — without loading anything into context.
1. Run `grep -rni --include='*.md' '<query>' .recall/` (add obvious variants/synonyms of the query as extra greps if the literal term misses).
2. Report pointers only: `file_path:line_number` plus the single matched line, grouped by feature, newest sessions first. Cap at ~20 hits; say how many were cut.
3. Load NOTHING. The user picks which pointer to open. This is the sanctioned route into L3 — the hit itself is the "explicit need" the hard rules require, but only for the specific file the user picks.

### `/recall doctor`
Read-only integrity check of `.recall/`. Report findings; offer fixes, apply only on confirmation.

Checks, in order:
1. **Orphan directories** — `features/<f>/` on disk with no entry in `tree.md`.
2. **Phantom entries** — features listed in `tree.md` with no directory on disk.
3. **Date drift** — `last:` in `tree.md` disagrees with `overview.md`'s "Last touched" or with the newest session filename date.
4. **Dead globs** — a feature's `paths:` patterns match zero files in the repo (code moved or was deleted; feature may need re-pathing or archiving).
5. **Archive debt** — features with > 5 session files that were never archived.
6. **Recent drift** — entries in `recent.md` that don't correspond to an existing session file.
7. **Unreadable files** — any `.recall/` file that's empty or missing its expected sections.

Output one line per finding: `<check>: <feature/file> — <problem> — <proposed fix>`. If everything passes, say so in one line. Never fix silently.

## Auto-resume behavior

When this skill is triggered by an open-ended status question in a directory containing `.recall/`:
1. Read L1 (`tree.md` + `recent.md` + `project.md`) — that's it.
2. Run the staleness check from `/recall status` and include any warnings.
3. Greet with the most recent session as a one-liner and offer to continue.
4. Wait for the user to commit to a direction before loading any L2.

When the user mentions a feature by name, or asks about something that clearly maps to a feature in `tree.md`, load that feature's L2 — but only that one, not its siblings, and not its session detail.

## Hard rules

- **Never auto-load L3.** Always wait for explicit need.
- **Never load more than one feature's L2 at once unless the user has explicitly pivoted.**
- **Always confirm feature assignment before dumping.**
- **`overview.md` "Current state" is last-write-wins.** Replace, don't append. Stale state must not accumulate.
- **Session files are append-only history.** Never edit a past session file; archive it instead.
- **When `.recall/` doesn't exist, do not create it silently.** Always ask the user first.
- **Don't dump without something worth dumping.** If the conversation didn't change project state (just reading code, asking questions), say so and skip the write.

## What this is NOT

- Not a replacement for `git log` / `git blame` — those are authoritative for code history.
- Not user-level memory (preferences, role) — that lives in `~/.claude/projects/.../memory/`.
- Not a documentation system — `overview.md` is for *current state and open threads*, not how-to docs.
