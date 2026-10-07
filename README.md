# recall

**The simplest persistent agentic memory for your workflow.** Markdown files in your repo, no database, no server, no embeddings. Claude Code picks up where you left off and loads about 1k tokens to do it.

```
you:    what was I working on?
claude: You were on billing/stripe-webhooks: retry logic works, signature
        check still open. Continue, or something else?
```

## Why

Agents forget everything between sessions. The usual fixes cost you either way:

- **Re-explain** the project every time (your time, plus tokens spent re-reading code).
- **One big handoff file** that grows forever and gets reloaded in full on every resume, including 90% that has nothing to do with today's task.

recall keeps memory as a **tree of features** and loads it in **tiers**. A resume reads a small index. It only goes deeper when you do.

## Install

```bash
git clone https://github.com/0xS4N4TG/recall ~/.claude/skills/recall
```

That is the whole install. Start Claude Code in any project and run `/recall dump` at the end of a session, or `/recall` at the start of the next one.

## How it works

Three load tiers:

| Tier | Files | Loaded when | Size |
|---|---|---|---|
| **L1** | `tree.md`, `recent.md`, `project.md` | every resume | ~1-4 KB |
| **L2** | `features/<f>/overview.md`, `decisions.md` | you mention that feature | ~1-3 KB |
| **L3** | `features/<f>/sessions/*.md` | you explicitly deep-dive | as large as the session |

L3 is never auto-loaded. Most resumes stop at L1.

```
.recall/
├── config.md          # personal (gitignored) or shared (committed)
├── tree.md            # feature index, mapped to file globs
├── recent.md          # last 3 sessions across all features
├── project.md         # stack, conventions, entry points
└── features/<feature>/
    ├── overview.md    # current state + open threads (last-write-wins)
    ├── decisions.md   # why things are the way they are
    ├── archive.md     # folded-up old sessions
    └── sessions/YYYY-MM-DD-<slug>.md
```

## Features

| Command | What it does |
|---|---|
| `/recall` | Where you left off, from L1 only. Also warns if code changed since the last dump. |
| `/recall load <feature>` | Pull one feature's overview and decisions into context. |
| `/recall dump [feature]` | Save the session. Routes to the right feature by the files you touched, asks before writing. |
| `/recall tree` | Show the whole feature tree. |
| `/recall new <feature> [parent]` | Add a feature, with `paths:` globs proposed from your session. |
| `/recall search <query>` | Grep all memory, L3 included, and return `file:line` pointers without loading anything. |
| `/recall doctor` | Read-only integrity check: orphan features, dead globs, date drift, archive debt. Fixes only on your OK. |

It also triggers on plain language: "continue where we left off", "save context", "wrap up".

### What makes it different

- **Feature-segmented.** A session that touched `auth/` and `billing/` is split into two dump files, one per feature. Routing uses file globs in `tree.md`: one feature with 70% or more of the touched files wins, close calls split, no match asks you.
- **Stale-memory detection.** Each dump records the git branch and short SHA. On resume, recall checks `git log` against each active feature's paths and flags features where code moved after the last dump. When memory and git disagree, git wins.
- **Last-write-wins state.** `overview.md` "Current state" is replaced on every dump, never appended, so the L2 layer stays the same size no matter how long the project runs.
- **Append-only history.** Past session files are never edited. Past 5 sessions per feature, recall offers to fold the oldest into `archive.md` bullets.
- **Decisions log.** One-line "why" entries per feature, so Claude stops re-litigating choices you already made.
- **Personal or shared.** First dump asks: gitignore `.recall/` (just you) or commit it (teammates inherit your context).
- **Never silent.** It won't create `.recall/` without asking, won't write without confirming the feature, and skips the dump when the session changed nothing.
- **Plain markdown.** Readable, diffable, greppable, editable by hand. No lock-in.

## Token savings

Worked example, not a benchmark. Assumptions: 12 features, 5 sessions each, ~3 KB per session, 4 characters per token.

| Approach | Loaded on resume | Tokens |
|---|---|---|
| One flat handoff file (all history) | 12 × 5 × 3 KB = 180 KB | **~45,000** |
| recall, resume only (L1) | ~3 KB | **~750** |
| recall, resume + one feature (L1 + L2) | ~3 KB + ~3 KB | **~1,500** |

That is roughly **97% fewer tokens per resume** in this scenario, and the gap widens as the project grows: the flat file grows with every session, while L1 grows only with the number of features and L2 stays flat because the state is overwritten. The tiers also keep the context window free for the work, which matters more than the raw count.

Your numbers will differ. Measure it yourself: `wc -c .recall/tree.md .recall/recent.md .recall/project.md` and divide by 4.

Savings compound across the loop too. With a recall memory in place, Claude doesn't need to re-read files to rediscover what you already decided.

## Not a replacement for

- `git log` / `git blame`: authoritative for code history.
- User-level preferences: those belong in Claude's own memory.
- Documentation: `overview.md` holds current state and open threads, not how-to docs.

## License

MIT
