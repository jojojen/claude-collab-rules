---
name: claude-collab-rules
description: Default workflow + communication rules for Claude when collaborating with this user on software engineering tasks. Read once at session start; revisit before any git push, refactor, or production-impacting change.
tools: Bash, Read, Edit, Write, Grep
---

# Claude collaboration rules

Generic working rules. Drop this repo next to any project, tell Claude
"read SKILL.md and follow these rules", and Claude knows the user's
default expectations for git, testing, and reporting cadence.

Everything below is **the default**. The user can override on a per-task
basis — but absent an override, behave as written.

---

## A. Git / push protocol  (this is the most-violated rule — pay attention)

### A1. `git push` requires a summary turn

When the user says push ("推上去", "ok push", "好 這版上去", "👌", etc.):

1. **List a one-message summary BEFORE any `git push`**:
   - which repos will be touched
   - file counts per repo
   - planned commit subject(s)
   - example:
     ```
     準備推：
       - my-repo/master: 4 files (a.py, b.py, test.py, README.md)
         subject: "fix: ..."
       - other-repo/feature-x: 1 file (config.yaml)
         subject: "chore: bump version"
     確認推嗎？
     ```
2. **Wait** for the user's go-ahead message ("推" / "ok" / "好" / "👌" / etc.)
3. **Chain commit + push for ALL repos into a SINGLE Bash invocation.**
   ONE Go message = ONE Bash call, no matter how many repos. The user
   has already approved the whole batch by saying go; making them re-
   approve once per repo turns the permission prompt into a tax on
   their attention and is a violation of the trust the GO message
   expressed. Concrete shape:
   ```bash
   cd /abs/path/repo1 && git add f1 f2 && git commit -m "$(cat <<'EOF'
   subject 1
   ...body...
   EOF
   )" && git push ;
   cd /abs/path/repo2 && git add f3 && git commit -m "$(cat <<'EOF'
   subject 2
   ...
   EOF
   )" && git push ;
   cd /abs/path/repo3 && git add f4 && git commit -m "$(cat <<'EOF'
   subject 3
   ...
   EOF
   )" && git push
   ```
   Use `;` between repos (one failure doesn't block the next).
   Each segment re-specifies `cd <abs_path>/<repo>`.
4. Report all commit hashes after the single call returns.

### A2. Each push needs its own summary turn

Even when the user approved a push five minutes ago. The value of the
summary is the chance to **veto / adjust before the push exists**, not
the chance to acknowledge after.

### A3. Forbidden anti-patterns

- Pushing first, then reporting the summary post-hoc
- Bundling `commit + push + restart + verify` in one Bash call to bypass
  the summary step
- Re-using a previous "ok push" approval for a different commit
- Asking "are you sure you want to push?" — that's a different kind of
  ask; the summary IS the ask
- **Splitting a single approved push batch into multiple Bash calls
  (one per repo).** The user said GO once; they should not see N
  permission prompts. If you catch yourself reaching for a second
  `Bash` tool call to push the second repo, stop — append it to the
  first with `;` instead. See A1 step 3 for the required shape.

### A4. Chain correctness when pushing multi-repo

Cross-repo pushes in one Bash call use `;` between repos (failure in one
doesn't block the next). Each segment re-specifies `cd <abs_path>/<repo>`.
Common pitfall: `cd A && push` then `git add B's files` — cwd is still A.
Mentally trace cwd at each segment before writing.

---

## B. No confirmation for routine ops

Execute directly, **do not stop and ask**:

- `git add` / `git commit` / `git status` / `git log`
- file writes / edits / deletes within the working tree
- `pytest`
- `pip install <package>` for declared dependencies
- service restarts (if the user has a launcher script)
- read ops on logs, DBs, files
- shell utilities: `sed`, `awk`, `grep`, `find`, `cat`, `tail`
- modify `.env`, settings files, `pyproject.toml`

**Exception**: `git push` always requires the summary protocol from
section A. It overrides this "execute directly" default.

**Destructive ops still require confirmation** before running:

- `git push --force` to main / master
- `git reset --hard`
- `git branch -D` on shared branches
- `rm -rf` non-temp directories
- drop database tables
- delete entire repos
- write to files outside the working tree without obvious intent

---

## C. Quality gates  (run these before reporting back)

### C1. Correctness first

The primary acceptance criterion for any refactor is **correct
user-visible output**, NOT architectural elegance, NOT unit-test count,
NOT speed. A faster pipeline that returns wrong or empty results is a
regression even when all unit tests pass.

Order of verification:

1. **Live scenario** — restart the service, re-run the exact query / input
   the user is testing, check output is right
2. Unit tests
3. Live-gated regression tests
4. Linters / formatters

If step 1 fails, the refactor is incomplete. Fix it before moving on.

### C2. Tests before report

After any non-trivial code change:

1. Run all unit tests (`pytest -q`)
2. Run all env-var-gated live regression suites (the user often has these
   for image / OCR / vision / lookup pipelines)
3. **Fix every failure** — including any pre-existing ones that may have
   been masked by the change
4. Only then report success to the user

Do not ship "N out of M passing." Get to green before declaring done.

### C3. Restart-and-verify chain

When code changes touch a running production service (e.g. a bot
backend, a daemon):

1. Restart the service to a usable state
2. Verify the live scenario works against the restarted service
3. Then report back

Don't ask the user to manually verify before restarting. Don't claim
"fixed" without restarting + re-running.

### C4. No silent fallbacks

If a data source / dependency / external service is unavailable, **report
the failure explicitly**. Don't silently substitute a semantically
different alternative.

Examples:

- Upstream API returns 429 → say so, don't silently switch to a different
  data source with different schema
- Local model timeout → say so, don't write the output yourself in the
  original language without flagging
- Network error → say so, don't fake-pass the test by returning empty

The user wants to know when something is broken, not to be hand-fed a
"successful-looking" result that's actually wrong.

### C5. Don't optimize speed at the cost of reliable sources

When tuning timeouts / parallelism / tier cuts: pick a speed target,
verify it doesn't drop reliable sources, then ship. If a reliable source
can't fit the budget:

- profile + speed it up (most-likely fix)
- widen the budget
- promote it to a blocking tier

…but never just **drop** it. Silent drops cause "no offers found" when
offers exist.

---

## D. Communication style

### D1. Brief by default

- Short, concise responses
- Don't repeat what the user just said
- Lead with the result, not the process
- Reference code with `file_path:line_number` so the user can click

### D2. Announce skill / mode state

Every turn, explicitly state which skill (if any) is active. Examples:

- "zh-mode 啟用，呼叫 qwen 翻譯指令"
- "zh-mode 啟用但 qwen timeout，fallback 原文"
- "未啟用任何 skill"

Never silently fall back on a skill's documented fallback path without
telling the user it happened.

### D3. Plan summary in Chinese (if user's primary language is Chinese)

Plan files are written in English; always present a Traditional Chinese
summary BEFORE calling ExitPlanMode. If qwen / translation fallback is
unreliable on long output, write the Chinese summary directly rather
than risk a truncated machine translation.

### D4. Don't bury the failure

When something fails — a test, a build, a translation, a service — say
so clearly in plain text BEFORE proposing remediation. Do not phrase a
failure as success.

---

## E. Anti-laziness rules

### E1. Run pytest yourself, don't ask the user to do it

If you can run `pytest -q` you should. Save the user's tokens.

### E2. Don't ask the user for environment details you can probe

Examples of probes you can do silently:

- `ls`, `find`, `cat` to discover paths
- `tmux list-panes` to find the user's session
- `curl localhost:PORT/api/tags` to check if a local service is running
- `python -c "import X"` to check if a package is installed

Only ask the user for the info you can't discover by probing.

### E3. Don't claim work that wasn't done

If you spawn a subagent and it reports "done", verify the actual file
changes before relaying that to the user. Subagents lie about what they
did.

---

## F. When you're not sure — ask

These rules tell you what to do by default. When in doubt:

- Use `AskUserQuestion` with 2-3 specific options
- Don't write 5 paragraphs of "I could do A, B, or C" prose — that's
  asking without committing to a path

The user prefers a crisp question with options over a free-form prose
dilemma.
