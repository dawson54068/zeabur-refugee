# Contributing to zeabur-refugee

Thanks for your interest in improving this guide. This repository is a Claude Code skill and a paste-to-your-LLM migration playbook for people leaving Zeabur after a credential exposure incident.

By participating, you agree to follow the [Code of Conduct](CODE_OF_CONDUCT.md).

## Getting started

This project is mostly Markdown plus a packaged `.skill` archive. There is no dependency install step for normal documentation changes.

1. Fork the repository and clone your fork.
2. Read `README.md` and `SKILL.md` so your change matches the existing flow.
3. Make the smallest focused change that solves the problem.
4. If you change skill behavior, update the relevant reference file under `references/`.
5. If you change evaluation expectations, update `evals/evals.json`.

## Local checks

Before opening a pull request:

1. Search the files you changed for leftover template markers or unfinished notes.
2. Run a worktree-only secret scan:

```bash
bash ~/.claude/skills/open-source-prep/scripts/scan_secrets.sh . --worktree-only
```

If you do not have the `open-source-prep` skill installed, at minimum run a local secret scan tool you trust before submitting.

## Writing guidelines

- Prefer Traditional Chinese for user-facing rescue instructions.
- Use Taiwan technical community wording where possible.
- Keep steps short and action-oriented; avoid giving the user a huge task list at once.
- Never ask users to paste API keys, database URLs, `.env` values, or credentials into chat.
- When mentioning platform pricing, limits, or product behavior, include official source links and dates where useful.

## Pull request guidelines

- Describe what changed and why.
- Keep pull requests focused; separate platform-specific updates from broad flow changes.
- For security-sensitive guidance, explain the failure mode your change prevents.
- Confirm there are no real secrets, credentials, or private user data in examples.

## Reporting bugs and requesting features

Open a GitHub issue for ordinary bugs, outdated platform instructions, missing migration targets, or wording improvements.

For security issues in this repository, follow [SECURITY.md](SECURITY.md) instead of opening a public issue.
