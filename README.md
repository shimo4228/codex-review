# codex-review

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/codex-review)

> **Retired on 2026-09-27. This repository is a public record of the skill as it was last published; it is no longer synced or developed.** For cross-model review from Claude Code, the author now uses OpenAI's official Codex plugin ([openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc)): `/plugin marketplace add openai/codex-plugin-cc`, then `/plugin install codex@openai-codex`. Why it was retired is under [Status and sync](#status-and-sync).

An [Agent Skill](https://agentskills.io/specification) that adds **one cross-model seam** to a code review chain: a thin, read-only wrapper around the [OpenAI Codex CLI](https://github.com/openai/codex) (`codex review`) so a *different model family* reviews your diff and catches blind spots that an author and a same-model reviewer share.

## Install

Both routes install the same frozen copy (`skills/codex-review/`: a `SKILL.md` plus the scripts); use the first for Claude Code, the second if you install skills through SkillsMP (a third-party Agent Skills marketplace). Whichever route you take, the skill has to end up at `~/.claude/skills/codex-review/`, because `SKILL.md` calls its scripts at that fixed path; a route or agent that installs it elsewhere will not find them. For a maintained route, use the official plugin above.

Either route requires the [Codex CLI](https://github.com/openai/codex) installed and authenticated (`codex login` or `codex doctor`); if the CLI is missing, the skill fails fast with a fallback message (see Failure Modes). Running a review sends your diff, or the plan file you pass, to OpenAI through the Codex CLI, and Codex may read (but not write) files in the repository it runs in; it runs under your own Codex login, so any usage limits or charges are those of your OpenAI account.

### Claude Code

```bash
git clone https://github.com/shimo4228/codex-review.git
# Copy into your global skills directory
mkdir -p ~/.claude/skills
cp -r codex-review/skills/codex-review ~/.claude/skills/codex-review
```

### SkillsMP

```bash
/skills add shimo4228/codex-review
```

## How It Works

This is a **decorrelation** seam (a reviewer whose mistakes do not line up with yours), not a throughput tool. Use your agent's own
subagents/parallel workflows for throughput; reach for a second *model* only
where a different model's judgment adds something a same-model reviewer can't.

Each invocation in the table below runs the installed script with the arguments you pass after `/codex-review` (`$ARGUMENTS`):

```
bash ~/.claude/skills/codex-review/codex-review.sh $ARGUMENTS
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

**Read-only by construction, in both the arguments and the config.** The script uses `codex review`
(never `codex exec -p yolo`) and only forwards the allowlisted flags above, plus
`--prompt "<text>"` for a prompt that starts with `-`. Any
other flag (a future `--write`, a user-supplied `-c` override) is rejected with
`exit 64`. The argv allowlist alone is not enough, though: `codex review` has no
`--sandbox` flag and inherits `~/.codex/config.toml`, where settings such as
`sandbox_mode = "workspace-write"` or Codex execpolicy `.rules` (command
pre-approvals) allowing `git push` would quietly turn a "read-only" review into
a writing one. The script therefore always pins `-c sandbox_mode="read-only" -c
approval_policy="never"` itself. The invariant is enforced in the script, not
assumed of the Codex CLI or of your config.

### Plan-stage premise challenge

The same seam, one step earlier: Codex challenges the premises of a plan instead of reviewing a diff. Before a design is frozen, hand Codex the
design packet (the plan file: premises / goal / chosen approach / rejected
alternatives / expiry) and ask for **refutations, missing constraints, and
cheaper alternatives — never a design**:

```
bash ~/.claude/skills/codex-review/codex-plan-challenge.sh --plan <packet.md> [-m <model>] [--focus "<one line>"]
```

Codex runs as `codex exec --sandbox read-only --ephemeral --ignore-user-config
--ignore-rules -c approval_policy="never"` with read-only access to the repo so
it can check the packet's premises against the code. It returns `REFUTE` /
`MISSING` / `ALTERNATIVE` findings and one `VERDICT:` line (`premise-hole` /
`alternative-exists` / `no-objection`) — no scores, no blueprint.

The point is to decorrelate **divergence only**: a different model family is
good at finding the blind spots in your premises, and bad as a co-author of
your design. Design authority (convergence) stays with the calling session;
findings are adopted or discarded one by one, never blended. Keep it to one
outside voice per design — once the main loop becomes an arbiter between
several designs, authorship is gone.

## After Running — fold, don't dump

Codex output is **untrusted input to a review decision the calling agent owns**,
not a verdict to relay verbatim:

- Verify each finding before acting on it — drop what you can disprove, keep
  what you confirm.
- Treat a confirmed critical finding as a stop signal, same as any other
  reviewer in the chain.

## When to Ask for It

The skill is opt-in only: it runs when you invoke it (`/codex-review`, or "codex review" / "second opinion on this diff") or when another skill explicitly wires it in as a review step. Nothing in the skill's settings blocks automatic use; its description tells the agent not to start it unprompted. Cases where you might ask for it:

- Before committing a non-trivial feature or fix; if you ask for it alongside
  your other reviewers, run it in parallel with them and handle its findings as in After Running
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

## Status and sync

The skill was published from the author's live Claude Code harness through `scripts/sync-from-local.sh`. Keeping this wrapper in step with Codex CLI changes had become a running cost, so on 2026-09-27 the harness switched to the official Codex plugin and deleted its copy (decision record: [harness ADR-0084](https://github.com/shimo4228/claude-harness/blob/main/docs/adr/0084-retire-codex-review-skill-for-official-codex-plugin.md)). The script now has nothing to sync from. The files here are the last published version: release 1.1.0 (2026-08-22, see CHANGELOG) plus two later `SKILL.md` syncs, the last on 2026-09-02.

## More from the author

- **[I Built a Skill for Easy Codex Reviews from Claude Code](https://dev.to/shimo4228/i-built-a-skill-for-easy-codex-reviews-from-claude-code-4h89)** ([日本語](https://zenn.dev/shimo4228/articles/codex-review-cross-model-decorrelation)): why the author added a second model family to the review chain, and why the skill stayed read-only and narrow.
- **[Can Six-Month-Old AI Code Survive Today's Review? A 25-Bug Triage](https://dev.to/shimo4228/can-six-month-old-ai-code-survive-todays-review-a-25-bug-triage-4o6d)** ([日本語](https://zenn.dev/shimo4228/articles/ai-code-half-year-audit)): this skill used on a real codebase; Codex's findings were strongest on external contracts such as API pricing and import formats.
- **[claude-harness](https://github.com/shimo4228/claude-harness)**: the author's daily-use Claude Code harness, published, with the ADRs that introduced this skill and later replaced it with the official Codex plugin.
- **[llm-as-judge](https://github.com/shimo4228/llm-as-judge)**: designs LLM judges with binary checks as evidence and one named verdict, for when you want a model's review output to end in a decision rather than a score.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with the five long-running projects and their DOIs.

## License

MIT. See [LICENSE](LICENSE).

<details>
<summary>For tools and AI assistants</summary>

codex-review is a retired Agent Skill for Claude Code that wraps the OpenAI Codex CLI in a read-only script, so that a different model family reviews a diff (`codex review`) or challenges the premises of a design plan (`codex exec`) and catches blind spots that the author and a same-model reviewer share. It is kept as a public record for developers who want a cross-model review step in a coding agent's workflow.

It exists because same-model agents scale throughput and context, while a different model scales judgment decorrelation, and the two do not substitute for each other. The skill therefore stayed narrow (read-only, one CLI, one review seam) instead of growing into a general multi-model orchestrator. It was retired from the author's harness on 2026-09-27 (harness ADR-0084) because following Codex CLI changes had a running cost; the author's harness now uses OpenAI's official Codex plugin for Claude Code (`codex@openai-codex`).

Canonical facts: MIT license; a `SKILL.md`, two Bash scripts (`codex-review.sh`, `codex-plan-challenge.sh`) and their test scripts under `skills/codex-review/`; last release 1.1.0 (2026-08-22) per CHANGELOG, plus two later `SKILL.md` syncs through 2026-09-02. Maintainer: shimo4228. Status: frozen record, no longer synced from the author's harness or developed. Requirements: the Codex CLI installed and authenticated (`codex login`), which sends the reviewed diff or plan to OpenAI under the user's own Codex login, with that account's usage limits or charges, and lets Codex read (not write) files in the repository it runs in; `git`; Claude Code or another Agent Skills-compatible agent that loads the skill from `~/.claude/skills/codex-review/`, the path `SKILL.md` hardcodes for its scripts. The script forwards only allowlisted flags (anything else exits 64) and always pins `-c sandbox_mode="read-only" -c approval_policy="never"`, because `codex review` has no `--sandbox` flag and would otherwise inherit `~/.codex/config.toml`.

Example: `/codex-review --uncommitted` runs `bash ~/.claude/skills/codex-review/codex-review.sh --uncommitted`, which asks Codex to review staged, unstaged and untracked changes with its built-in review instructions; the calling agent then verifies each finding, drops what it can disprove and treats a confirmed critical finding as a stop signal. Exit 3 means the Codex CLI is missing and exit 4 means the directory is not a git repository. `codex-plan-challenge.sh --plan <packet.md>` instead returns `REFUTE` / `MISSING` / `ALTERNATIVE` findings and one `VERDICT:` line (`premise-hole`, `alternative-exists` or `no-objection`).

Links: [skills/codex-review/SKILL.md](skills/codex-review/SKILL.md) is the skill, [CHANGELOG.md](CHANGELOG.md) the history, [llms.txt](llms.txt) and [llms-full.txt](llms-full.txt) the machine-readable summary and reference, and [ADR-0084](https://github.com/shimo4228/claude-harness/blob/main/docs/adr/0084-retire-codex-review-skill-for-official-codex-plugin.md) in claude-harness the retirement decision. The author's hub is [shimo4228/shimo4228](https://github.com/shimo4228/shimo4228).

</details>
