# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] — 2026-07-05

Initial release as a standalone skill repository.

### Added

- `skills/codex-review/SKILL.md` — read-only wrapper around the OpenAI Codex CLI (`codex review`) that adds a cross-model decorrelation seam to a code review chain: scoped modes (default branch-vs-base, `--uncommitted`, `--base`, `--commit`), prompt-driven mode, model selection, an enforced read-only flag allowlist, and fold-don't-dump handling of Codex's output.
- `skills/codex-review/codex-review.sh` — the wrapper script itself.
- `skills/codex-review/test-codex-review.sh` — script test suite.
- `scripts/sync-from-local.sh` — one-way export from the live Claude Code harness; the harness copy is canonical, this repository is the publication mirror.

### Notes

- Previously published only inside the author's aggregated [claude-harness](https://github.com/shimo4228/claude-harness) repository. Extracted here so the skill is installable on its own.
