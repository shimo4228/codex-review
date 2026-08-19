# codex-review

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/codex-review)

An [Agent Skill](https://agentskills.io/specification) that adds **one cross-model seam** to a code review chain: a thin, read-only wrapper around the [OpenAI Codex CLI](https://github.com/openai/codex) (`codex review`) so a *different model family* reviews your diff and catches blind spots that an author and a same-model reviewer share.

## Install

### Claude Code

```bash
# Copy into your global skills directory
cp -r skills/codex-review ~/.claude/skills/codex-review
```

### SkillsMP

```bash
/skills add shimo4228/codex-review
```

Requires the [Codex CLI](https://github.com/openai/codex) installed and authenticated (`codex login` or `codex doctor`) — the skill fails fast with a fallback message if it isn't.

## How It Works

This is a **decorrelation** seam, not a throughput tool — use your agent's own
subagents/parallel workflows for throughput; reach for a second *model* only
where a different model's judgment adds something a same-model reviewer can't.

```
skills/codex-review/codex-review.sh $ARGUMENTS
```

| Invocation | Scope |
|---|---|
| `/codex-review` | current branch vs auto-detected base (`main`/`master`/…) — PR-style |
| `/codex-review --uncommitted` | staged + unstaged + untracked — pre-commit |
| `/codex-review --base <branch>` | vs an explicit base branch |
| `/codex-review --commit <sha>` | a single commit |
| `/codex-review -m <model>` | pick a Codex model (combine with any row) |
| `/codex-review "focus on the auth changes"` | prompt-driven review of the working tree |

**Scope and prompt are mutually exclusive** (a codex-cli constraint, ≥ 0.142): a
*scoped* review (`--uncommitted` / `--base` / `--commit`, or the default) uses
Codex's built-in review instructions and takes no custom prompt; a bare prompt
drives a working-tree review with no scope flag. `-m/--model` may accompany
either.

**Read-only by construction.** The script uses `codex review` (never `codex
exec -p yolo`) and only forwards the allowlisted flags above — any other flag
(a future `--write`, a `-c` config override) is rejected with `exit 64`. The
invariant is enforced in the script, not assumed of the Codex CLI.

## After Running — fold, don't dump

Codex output is **untrusted input to a review decision the calling agent owns**,
not a verdict to relay verbatim:

- Verify each finding before acting on it — drop what you can disprove, keep
  what you confirm.
- Treat a confirmed critical finding as a stop signal, same as any other
  reviewer in the chain.
- Run it in parallel with same-model reviewers, then merge verdicts.

## When It Triggers

- Before commit on a non-trivial feature or fix, alongside your other reviewers
- When you want a second opinion from a non-Claude model on a diff
- High-stakes or error-prone changes where decorrelated review pays off
- High-stakes **prose** diffs before publishing (README, paper, article) —
  prompt-driven mode with writing-focused instructions; scoped modes run
  Codex's code-review instructions, which fit prose poorly

Skip it for trivial edits, throwaway scripts, or when Codex isn't authenticated.

## Failure Modes

- `exit 3` — codex CLI missing / not installed → continue with same-model reviewers only
- `exit 4` — not inside a git repository → cannot diff
- Auth not configured → `codex review` errors; run `codex login` (or `codex doctor`)

## Syncing from the harness

The canonical copy of this skill lives in the author's live Claude Code harness. This repository is a one-way publication mirror:

```bash
scripts/sync-from-local.sh --dry-run   # report differences only
scripts/sync-from-local.sh             # apply to working tree (never commits)
```

## About this skill

This skill implements a single cross-model seam in a broader multi-agent
orchestration principle: same-model agents scale throughput and context, a
*different* model scales judgment decorrelation — the two axes don't
substitute for each other, so this skill deliberately stays narrow (read-only,
one CLI, one seam) rather than growing into a general multi-model orchestrator.

It is maintained by [@shimo4228](https://github.com/shimo4228), whose research
lines are the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle)
([DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)),
[Contemplative Agent](https://github.com/shimo4228/contemplative-agent)
([DOI 10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118)) —
autonomous agents grounded in four contemplative axioms — and
[Agent Attribution Practice (AAP)](https://github.com/shimo4228/agent-attribution-practice)
([DOI 10.5281/zenodo.19652013](https://doi.org/10.5281/zenodo.19652013)) —
harness-neutral ADRs on accountability distribution.

## License

MIT
