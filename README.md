# claude-collab-rules

Default working rules for Claude when collaborating with me on software
engineering tasks. Drop this repo next to any project, point Claude at
`SKILL.md`, and Claude knows my git / testing / reporting expectations
without me having to retype them.

## Usage

### Quick start

From any project directory:

```bash
git clone https://github.com/jojojen/claude-collab-rules.git
# then tell Claude (in a new session):
# "Read claude-collab-rules/SKILL.md and follow those rules for this session."
```

### Or as a shared sibling repo

```bash
cd ~/work
git clone https://github.com/jojojen/claude-collab-rules.git
# in each project: tell Claude "the rules are at ../claude-collab-rules/SKILL.md"
```

## What's in here

| File | Purpose |
|------|---------|
| `SKILL.md` | The rules themselves. Sections A-F cover git/push protocol, no-confirmation defaults, quality gates, communication style, anti-laziness, and "when to ask". |
| `README.md` | This file. Human overview + how to use. |

## What these rules optimize for

- **Low friction**: Claude executes routine ops without asking.
- **High correctness**: Tests + live-scenario verification before any "done" claim.
- **Predictable git**: Push summary lists `{repo, file count, commit subject}` BEFORE the push happens, every time.
- **No silent failures**: If a tool / data source / model breaks, Claude says so before doing anything else.

## Updating

These rules came out of iterating with Claude on real projects. If a new
pattern keeps coming up (or an old rule needs strengthening), edit
`SKILL.md` and push. Future Claude sessions auto-pick up the new
behaviour as long as you remind Claude to re-read `SKILL.md` at session
start.

## License

Personal-use rules. No warranty. Adapt freely.
