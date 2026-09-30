# AGENTS.md — skill-openclaw

Project rules for this tree. This is a real file. Do not add a per-project `CLAUDE.md`.

## What this is

Maintenance home for the **openclaw** skill: the procedure an agent follows to
operate an OpenClaw gateway, on this machine or on another host.

| Piece | Value |
| --- | --- |
| Skill name | `openclaw` (frontmatter `name:`) |
| Slash command | `/openclaw` |
| Skill file | `SKILL.md` at this directory's root |
| Grok install | `~/.grok/skills/openclaw` → this directory |
| Claude Code install | `~/.claude/skills/openclaw` → this directory |
| Kimi Code install | `~/.kimi-code/skills/openclaw` → this directory |

The symlink's directory name is the skill name, `openclaw`. Human install
steps are in `README.md`.

## Where facts live

One home per fact.

- **Procedure** (how to reach a gateway, which commands, when to stop for approval): `SKILL.md`. Split into `references/` only when a section would bloat that file. Add `scripts/` only when a wrapper removes a real footgun. The OpenClaw CLI is the tool.
- **Instance facts** (SSH target, CLI prefix, unit name, URLs, token source, deployment notes): a profile in the deployment's own project, outside this repo. `instances/example.md` is only the template. This repo's gitignored `.env` holds the path (`OPENCLAW_INSTANCE` or `OPENCLAW_INSTANCES`). See `.env.example`.
- **Product schema** for the CLI and for hooks: `openclaw --help` on the gateway, then the docs linked from `SKILL.md`.
- **Secrets:** the profile's `hooks_token_source`, or the namespace named in that profile's notes. Leave the gateway `.env` unread and unprinted. Never commit a token.

Tracked files stay generic. No hostnames, addresses, personal names, account names, or `pass` paths. The public identity of this repo is the GitHub login `dlewis7444`. Commits use that login and a GitHub noreply address. Do not set a personal name or personal email on this repo.

## Git

GitHub repository `dlewis7444/skill-openclaw`, private. A push to `main` needs an explicit authorization in the request. Creating this private repository was that authorization for the first push.

## Check

After a procedure change, run a read-only check against the instance you mean.
Do not restart the gateway or edit its config as a test of this skill.

```bash
ssh <user@host> 'systemctl --user is-active <unit>'
ssh <user@host> '<cli_prefix> openclaw --version'
```

A local gateway drops the `ssh` wrapper.
