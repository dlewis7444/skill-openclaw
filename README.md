# openclaw

Workstation skill for operating an [OpenClaw](https://docs.openclaw.ai) gateway.
The same procedure covers a gateway on this machine or on another host. SSH
targets, the CLI `PATH`, the systemd unit, the Control UI, and hook credentials
live in an instance profile, not in the shared procedure.

Slash command: **`/openclaw`**. Optional argument: an instance name.

## Install

`SKILL.md` is at the repo root. Point the harness skill directory at this
repo. The symlink name is the skill name.

```bash
git clone git@github.com:dlewis7444/skill-openclaw.git ~/src/skill-openclaw
mkdir -p ~/.grok/skills ~/.claude/skills
ln -s ~/src/skill-openclaw ~/.grok/skills/openclaw
ln -s ~/src/skill-openclaw ~/.claude/skills/openclaw
```

Grok loads it from `~/.grok/skills/openclaw`.

Copy `instances/example.md` into the deployment's own project and fill it in.
Copy `.env.example` to `.env` and set `OPENCLAW_INSTANCE` to that file.
`.env` is gitignored. The profile does not live in this repo.

## Layout

| Path | Purpose |
| --- | --- |
| `SKILL.md` | Agent procedure |
| `instances/example.md` | Profile template |
| `.env.example` | Tracked shape of the workstation pointer |
| `.env` | Gitignored path to the local profile |

This repo is not installed into a gateway's `workspace/skills/`. Those skills
run inside the assistant. This one runs on the operator workstation.

## Publishing

The GitHub repository is private. Tracked files stay generic: no hostnames,
addresses, account names, or `pass` paths. The copyright line names the GitHub
login `dlewis7444` and no personal name. Visibility stays private until a
request to change it is explicit. Before that change, scan tracked files again
and confirm `.env` is untracked.
