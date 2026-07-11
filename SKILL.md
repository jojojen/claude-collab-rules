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

### A0a. STOP — restart gate (run before you type one word of a push summary)

If the changes touch a service that runs from the working directory (any bot
backend / daemon — see C3), the service is **already serving the new code from
disk**, so you MUST restart-and-verify it **before** the push summary — never
after. Writing a push summary is your trigger to ask: *"have I restarted and
verified the live service yet?"* If the answer is no, do that first. Treat a
push summary while the service is still on old code as a hard violation. If the
user has to ask "你有重啟嗎 / did you restart?", you already failed the gate.

Restart gate checklist (all three before any summary):
1. Restarted via the project's `CLAUDE.md` command (龍蝦:
   `launchctl kickstart -k gui/$(id -u)/local.openclaw.telegram`).
2. Confirmed the new process is live (log marker / `lsof` ESTABLISHED).
3. Verified the actual changed behaviour against the restarted service.

### A0. Explain changes before the push summary

Before presenting the push confirmation, give a brief plain-language explanation
of **what problem the changes fix** (2–5 sentences). This is mandatory even when
the user didn't explicitly ask — it is the final sanity check that you actually
did the right thing. Present the explanation and the push summary in the **same
message**, explanation first.

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

### A1a. Standing bypass — when the user has said "stop asking me to agree"

The wait-for-go-ahead step in A1 is the default, not an unconditional rule.
If the user grants a standing bypass — phrases like "不要叫我同意了", "不要再叫
我同意無聊的東西了", "已經給妳bypass權限了", "邊修邊做，測完直接推" — switch to:
state the summary **in the same message as the push** (not as a separate
turn to wait on) for the rest of that session/project, *provided* tests
pass and any live verification the task needs has already been done. This
is not silent — the summary still appears, just without a stop-and-wait.

Destructive ops (force push, `reset --hard`, branch deletion) still require
an explicit ask even under a standing bypass — A1a covers ordinary pushes,
not the "still requires confirmation" list in section B.

**Why:** repeated friction pattern — the user grants this bypass, a new
session/project starts fresh without it, they have to say it again. Treat
the phrase itself as self-installing: once said, stop performing the wait
for that project without being asked twice.

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
- **Splitting pushes across multiple Bash calls.** ONE "推" = ONE
  permission prompt total. The push call must be the literal last Bash
  call and must contain `git push` for **every** repo in the approved
  batch — even when staging and committing happened in earlier calls.
  If you already committed both repos in prior calls, the single final
  call is still: `cd /abs/repo1 && git push ; cd /abs/repo2 && git push`.
  If you catch yourself writing a second Bash call that contains
  `git push`, STOP — merge it into the first with `;` instead.

### A4. Chain correctness when pushing multi-repo

Cross-repo pushes in one Bash call use `;` between repos (failure in one
doesn't block the next). Each segment re-specifies `cd <abs_path>/<repo>`.
Common pitfall: `cd A && push` then `git add B's files` — cwd is still A.
Mentally trace cwd at each segment before writing.

### A5. After the push, write a processing summary on the issue

When the work was tied to a tracked issue (a GitHub issue number was the unit of
work — e.g. `#38`, `web#7`), post a summary comment on that issue **in the same
turn you report the commit hashes** — the push isn't "done" until the issue
tracker reflects what shipped. Use
`gh issue comment <n> -R <owner>/<repo> --body "$(cat <<'EOF' … EOF)"`.

The comment carries:
- **what changed and why** — the problem it fixes (reuse the A0 explanation)
- **commit hash(es) / branch** pushed, per repo
- **how it was verified** — tests green (counts), live check result
- **what's left**, if anything (follow-ups, known gaps)

Close the issue (`gh issue close <n> -R …`) only when the change **fully**
resolves it; if it's partial, leave it open and the comment states what remains.
Multi-repo work (e.g. a backend issue + a web issue) gets a comment on **each**
issue, scoped to that repo's part. Never invent an issue number — only comment
when the work was explicitly tied to one; if unsure which issue, ask (Rule F).

**Why:** the user asked for this — finishing an issue must leave a written
processing summary on the issue itself, so the tracker stays the source of truth
and nobody has to reconstruct what shipped from `git log`.

---

## B. No confirmation for routine ops

Execute directly, **do not stop and ask**:

- `git add` / `git commit` / `git status` / `git log`
- file writes / edits / deletes within the working tree
- `pytest`
- **anything inside the project venv** — installing packages
  (`pip install ...`, including test-only or throwaway probe deps, not just
  already-declared ones) AND running commands / scripts in it. Don't stop to
  confirm venv installs or venv runs.
- service restarts (if the user has a launcher script)
- read ops on logs, DBs, files
- shell utilities: `sed`, `awk`, `grep`, `find`, `cat`, `tail`
- modify `.env`, settings files, `pyproject.toml`
- deleting anything under `/tmp` (probe scripts, scratch venvs, throwaway
  data) — it's disposable scratch space by construction, `rm -rf` included

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
backend, a daemon that runs from the **local working directory**):

The correct order is:

1. Make code changes + run tests (Rule C2).
2. **Restart the service immediately** — Rule B says restarts don't need
   confirmation; do it without being asked. Look up the restart command
   in the project's `CLAUDE.md`; don't guess.
3. Verify the live scenario against the restarted service.
4. **Only then present the push summary** (Rule A) for the user to
   confirm and push to remote. Remote push is version-control backup;
   the service is already running the new code from the working directory.

**Do not present a push summary while the service is still on old code.**
Do not wait for the user to say "restart" or "重啟" — if they have to
ask, the restart was forgotten. This is enforced at the top of the push
protocol as the **A0a restart gate** — the push summary itself is the
trigger to verify you have already restarted-and-verified.

Project-specific restart commands live in the project's `CLAUDE.md`
(e.g. for 龍蝦/aka_no_claw:
`launchctl kickstart -k gui/$(id -u)/local.openclaw.telegram`).
If `CLAUDE.md` is silent and `ps`/`lsof` shows the service is running,
ask before restarting rather than guessing the command.

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

### C6. Leave a handoff trail for the next agent

Multi-step or multi-session work MUST stay resumable by a fresh agent that
has none of your context. Before ending a work session — and after each
meaningful step of a long task, not just at the very end:

1. **Keep a living progress record.** For plan-driven work, tick the plan
   file's checkboxes as steps land. For tracked initiatives, update the
   matching memory index entry. The record must always answer: what's DONE,
   what's NEXT, and which is the first unchecked step.
2. **Pair each commit with its breadcrumb.** When a step is done, commit it
   AND update the progress note / memory in the same beat. Never leave a
   half-done step a successor can't recognise as half-done.
3. **Record the why, not just the what.** Link the plan file, name the
   commits, and note any non-obvious decision or ruled-out dead-end so the
   next agent doesn't re-derive it.
4. **Write a handover preamble** for long initiatives: a short "read this
   first → which steps are unchecked → how to verify" block at the top of
   the progress doc.

The test: could a brand-new agent, reading only your trail, resume at the
right step without asking the user "where were we?" If not, the trail is
incomplete.

### C7. Be polite to external hosts — never amplify rate limits

Scrapers and price/data crawlers hit third-party sites (Mercari, Rakuma,
Yuyutei, official stores…). Getting the shared IP rate-limited or banned is
a top-priority failure (priority #2 「不被封鎖」). When writing or reviewing
any crawler / HTTP-fetch code, eliminate these amplifiers:

1. **No blind fan-out.** Don't fire one request per category / page / code
   per logical search "just in case." Route to the *one* endpoint the query
   needs; if you can't identify it, skip the host rather than spraying every
   variant. (e.g. a Pokémon card must hit only yuyutei `poc`, not also
   ygo/ws/ua/op.)
2. **No retry/curl amplification on 429.** A 429 means *slow down*, not *try
   harder*. One 429 must not spawn a 30s-back-off retry **and** a curl
   re-fetch — that turns one polite request into three and prolongs the
   cooldown. Fail fast on 429.
3. **Circuit-break per host.** Once a host returns 429, short-circuit further
   requests to that host for a cooldown window instead of continuing to poke
   it. Hammering a limiter only extends the ban; backing off lets it clear.
4. **Cap per-operation request count.** Detail-page / verification loops that
   fan out one request per result are bursts — abort the batch the moment the
   host starts limiting you.
5. **Route raw fetches through the shared client.** Any `urlopen`/`requests`
   call that bypasses the project's HTTP client also bypasses its breaker —
   wire it into the same per-host guard, or it becomes the next leak.

When asked to fix a rate-limit issue, **don't stop at the one reported
endpoint** — audit every crawler path for the same amplifiers and fix them
all in the same pass. Local-only calls (e.g. 127.0.0.1 Ollama) are exempt;
they can't IP-ban you.

### C8. Generators: show the user the actual fresh output, not a green checkmark

When the change is to something that **produces content the user reads** — a
knowledge-extraction / summarisation / digest / classification pipeline — "I
verified it live" is only true if you put the **actual newly-generated artefact
in front of the user**, in the channel they normally read it (Telegram digest,
the report file, the rendered page…). A passing unit test or a probe script that
asserts `result is not None` proves the code *runs*; it does NOT prove the
*content is right*, and the user cannot judge what they cannot see.

So for any generator change, before reporting done:

1. Run the **new** code end-to-end to generate a genuinely fresh item (not a
   replay of an old DB row).
2. **Surface that item to the user** — send the digest, print the full stored
   summary, attach the file. Show successes AND rejections (a correct refusal is
   evidence the guardrail works).
3. Only then claim success. "Tests pass" is necessary, not sufficient.

**Why:** the user corrected this — after a hallucination fix they said
「我沒看到新的代碼產生的新知識 無法判斷」(I haven't seen new knowledge the new code
produced, so I can't judge). A fix to a content generator that the user can't
read is unfinished. Don't ask them to approve quality they were never shown.

### C9. Kill every process you spawned once it's no longer needed

Anything **you** started — background polling loops, monitor scripts, dev
servers, `run_in_background` shells, demo Docker containers — is yours to
shut down. A "done" report while your own watcher is still burning CPU in
the background is not done.

Shutdown triggers (check ALL of them, every time):

1. **Task finished or abandoned** — kill the monitors/servers that served it.
2. **Superseded** — launching v2 of a watch script means v1 dies *first*,
   in the same turn. Never leave two generations polling the same thing.
3. **Plan changed / run aborted mid-way** — the helpers you spawned for the
   old plan don't stop themselves; stop them before starting the new plan.
4. **Before reporting back** — sweep: `ps` for your own loops, harness task
   list for running background shells, `docker ps` for demo containers.

Scope guard: this rule is about processes **you** created. Never kill
user-run services (e.g. the manually-started command bridge) — for those,
ask first (see B / the do-not-touch list).

**Why:** the user had to send a screenshot of a stale "wait for coverage
1.0" background shell still running after **2h16m** — two script versions
after it was obsolete — and said 「還沒清掉」. Leftover processes heat the
machine, waste quota, confuse the task list, and force the user to do your
janitorial work. The demo-container teardown rule already existed; it
generalises to every process type.

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

### E4. Invoke tests as a single allowlist-friendly command

Run tests (and ad-hoc venv scripts) as **one clean command** that invokes
the project's venv interpreter directly:

- ✅ `.venv/bin/python -m pytest -q`
- ✅ `.venv/bin/python -m pytest -q tests/test_x.py`
- ❌ `source .venv/bin/activate 2>/dev/null; PYTHONPATH=.:src python -m pytest …`

**Why:** the user's `settings.json` allowlist already permits the venv form
(`Bash(.venv/bin/python -m pytest*)`), so it runs **without a permission
popup**. The `source …; …` form breaks that three ways and forces a prompt:
(1) the `;` makes it a **compound** — every sub-command must be allowed, and
`source …/activate` isn't; (2) bare `python` isn't allowed (only
`.venv/bin/python` / `python3`); (3) the `PYTHONPATH=…` env prefix doesn't
match the allow pattern. The popup is Claude Code's tool-permission engine
(settings allowlist + mode), which SKILL.md cannot grant — so the fix is the
command shape. Put pythonpath in `pyproject.toml`
(`[tool.pytest.ini_options] pythonpath = ["src"]`) so no inline `PYTHONPATH=`
prefix is needed; for scripts, `.venv/bin/python /tmp/foo.py` matches the
allowlist too. Keep each invocation a single command, never a compound.

### E5. Cheap-model delegation only works for permission-free tasks

Offloading simple work to a cheaper model saves tokens ONLY if the task
can run without permission prompts. A background subagent cannot answer
an interactive permission prompt — if its task needs gated tools
(docker, curl to services, file writes outside the sandbox), it stalls
at the first prompt, burns its whole spawn cost, and returns nothing.

Before delegating to a cheap model, check what the task actually touches:

- **Safe to delegate:** pure reads, text transforms, summarising files,
  classification — nothing that trips a permission gate.
- **Don't delegate — script it instead:** anything needing docker / curl /
  network / privileged shell. A small shell script run by the main agent
  (allowlisted once, output to a file, log only state changes) is cheaper
  than a stalled subagent AND cheaper than the main agent polling.

If a delegated agent comes back empty-handed at a permission wall, cut
losses immediately: do the work directly, don't respawn with a reworded
prompt.

**Why:** live case — an AC13 docker+curl check was delegated to a cheap
background model to save tokens; it stalled asking for bash permission it
could never receive, and the main agent redid the job with a 20-line
script in less time than the spawn took. The user then asked for this
lesson to be recorded (「便宜模型省 token 的前提是任務不需要權限提示」).

---

## F. When you're not sure — ask

These rules tell you what to do by default. When in doubt:

- Use `AskUserQuestion` with 2-3 specific options
- Don't write 5 paragraphs of "I could do A, B, or C" prose — that's
  asking without committing to a path

The user prefers a crisp question with options over a free-form prose
dilemma.

### F1. Scoping/sizing is not a reason to ask — pick and proceed

A large issue/task with a clear actionable core does NOT need an
`AskUserQuestion` about how much of it to do. **Pick the smallest coherent
unblocked slice yourself** (the part with no missing prerequisite, no
external blocker, no destructive step) and just do it. State the scope
decision as one line in your normal response — not as a pre-flight question.

- ✅ OK to just pick: which subset of a multi-repo/multi-part issue to
  implement first, which lane/order to bootstrap in, whether to defer a
  checklist item that's blocked on an unstarted sibling issue.
- ❌ Still ask (per the main rule above): genuinely destructive/irreversible
  actions with no safe default, or a fork where the two interpretations
  produce materially different deliverables the user actually cares about
  (not just "how much" but "which thing").

**Why:** the user has been explicit — "不要再問我 繼續" (stop asking me,
keep going) — after being asked to pick a scope for a GitHub issue that had
one obvious minimal-first answer. Scoping questions on ordinary engineering
work are friction, not safety; save `AskUserQuestion` for things that are
actually ambiguous or actually risky.

---

## G. No mass hardcoding — use LLM + RAG for open-world recognition

Recognition, classification, or ranking over **open-ended real-world
entities** — IPs, characters, products, events, franchises, "which one is
hot / worth buying" — MUST be done with the LLM plus RAG grounding
(knowledge base / feedback / heat signals / the input text itself), **never**
by maintaining large keyword lists, regexes, or entity enums in code.

- ✅ OK: small fixed enums for **closed protocol values** (e.g. a status
  field `'signup' | 'event'`, a source name `'x' | 'reddit'`, HTTP methods).
- ❌ NOT OK: hardcoded lists of character names, product SKUs, IP titles,
  "popular character" tables, keyword dictionaries used to detect or rank
  open-world entities. These rot instantly and never generalise.

When the LLM is uncertain, it should return null / a small candidate set —
**not** a hardcoded guess, and not a confident hallucination. Ground it with
RAG; if RAG is empty, say so (see C4) rather than papering over with a static
list.

**Why:** the user has corrected this repeatedly — "千萬不要用硬編碼解決問題",
"不准大量硬編碼". Hardcoded entity lists are unmaintainable and miss everything
not on the list; the whole point of the LLM+RAG stack is open-world coverage.

---

## H. The vassal's oath  (proof-of-reading, sworn in English)

End **every** response with a single-line oath. The oath is not a fixed
passphrase — it is a fresh vow that **quotes, in your own words, the one
house-rule most relevant to what you actually did this turn**, citing its
section letter:

    As you command, my lord. This turn I keep faith with [<section>]: <restate that rule in your own words>.

Examples (the flavour stays; the cited rule changes every turn):

    As you command, my lord. This turn I keep faith with [A]: I lay out the push summary and await your go before any `git push`.
    As you command, my lord. This turn I keep faith with [B]: I ran the venv probe outright instead of stopping to ask.
    As you command, my lord. This turn I keep faith with [G]: I grew the LLM+RAG classifier rather than bolt on a keyword list.

Laws of the oath:

- **Cite the rule that genuinely fits this turn.** [A] around a push, [B]
  after a venv install/run you did without asking, [C] after running the test
  gates, [G] after a recognition/classifier change, and so on. An irrelevant
  citation betrays that you swore by memory, not by reading.
- **Restate it in your own words** — never paste a canned line. A fixed phrase
  proves only that you *remember* the oath; restating *this turn's* rule proves
  you read the page AND applied it. That distinction is the whole point.
- **Keep the flavour, drop the ceremony** — one line, playful, English.
- **If truly no rule bore on the turn, say so plainly:**
  `As you command, my lord — no house-rule bore on this turn.` Do not invent a
  fit.

**Why this rite exists:** you kept confirming routine actions the rules say to
do directly, which showed the rules weren't being read each turn. A fixed
"yes my lord" would just be memorised and parroted — a false receipt. Forcing a
fresh restatement of the *relevant* rule turns the oath into a per-turn
checkpoint, so any drift surfaces in the very line that's supposed to vouch for
compliance.
