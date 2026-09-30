---
name: openclaw
description: >
  Operate an OpenClaw runtime from this workstation: gateway status, logs,
  restart, the openclaw CLI, cron, the Control UI, hooks, and where runtime
  files live. The same procedure covers a gateway on this machine or on
  another host; host facts come from an instance profile outside this skill.
  Use whenever the user mentions OpenClaw, an OpenClaw gateway, or asks to
  talk to or manage an OpenClaw assistant, including from another project.
  Slash command: /openclaw.
argument-hint: "[instance]"
user-invocable: true
license: MIT
metadata:
  short-description: Operate an OpenClaw gateway
---

# OpenClaw

Operate an OpenClaw gateway from the workstation where this skill is installed.
Read one instance profile, then follow this procedure. Command details that
are not spelled out here come from `openclaw --help` on that gateway.

## Instance profile

Host facts live outside this repository, in the deployment's own project.
`instances/example.md` is the field shape. Copy it there. This repo does not
hold a real gateway.

A gitignored `.env` next to this file points at that profile. See
`.env.example`. Resolve a relative path from the physical directory of this
`SKILL.md` (the install symlink's target), not from the shell's current
directory. Absolute paths are used as written.

Resolve in this order:

1. A profile path the user gives.
2. `/openclaw <name>`, matched against profiles named in `.env` (`name`, SSH
   target, Control UI URL, or hooks URL).
3. `OPENCLAW_INSTANCE` when `.env` sets that one path.
4. Ask which profile to use. Do not invent a host.

`OPENCLAW_INSTANCES` is a colon-separated list of profile paths when this
workstation has more than one gateway. Use it instead of `OPENCLAW_INSTANCE`.

If `.env` is missing and the user gave no path, say so and stop before
touching a gateway.

The profile owns these fields:

| Field | Role |
| --- | --- |
| `name` | Short id, matched against `/openclaw <name>` |
| `transport` | `ssh` or `local` |
| `ssh` | `user@host` when transport is `ssh` |
| `cli_prefix` | Shell prefix that selects the working `openclaw`. Empty when the default binary is already the right one |
| `avoid_binary` | Optional path of a second `openclaw` that must not be run |
| `unit` | systemd user unit for the gateway |
| `runtime_root` | State directory on the gateway host. Default install is `~/.openclaw` |
| `control_ui` | Control UI URL, or `none` |
| `hooks_url` | Full URL for `POST /hooks/agent`, or `none` |
| `hooks_token_source` | How to read the bearer token at call time (`pass` path or other command). Never the token |
| `gateway_bind`, `gateway_port`, `trusted_proxies` | Listen facts, when the profile knows them |

Auth mode, reverse-proxy headers, runtime-skill names, and pointers to an
architecture writeup belong in that profile's notes.

## Run a command on the gateway host

Apply `cli_prefix` on every `openclaw` invocation when the profile sets one.
Skip `avoid_binary`.

SSH, with the remote command in single quotes so the prefix expands on the gateway host:

```bash
ssh <ssh> '<cli_prefix> openclaw <args>'
ssh <ssh> 'systemctl --user is-active <unit>'
ssh <ssh> 'journalctl --user -u <unit> -n 200 --no-pager'
```

`transport: local` runs those commands in the current shell, with no `ssh`.

When the question is what the gateway is doing right now, query the host.
If a command prints a token, leave it out of the chat and out of this repo.

## Gateway service

The unit is a systemd user unit. `systemctl --user` and `journalctl --user`
run as the runtime user, which is the SSH user when transport is `ssh`.

Read-only checks:

- `systemctl --user is-active <unit>`
- `journalctl --user -u <unit> -n 200 --no-pager`
- `openclaw gateway status`

Restarting is `openclaw gateway restart` or `systemctl --user restart <unit>`.
Both wait for approval below.

## CLI

Scheduled jobs go through `openclaw cron`, the same command family as
`openclaw automations`:

```bash
openclaw cron list
openclaw cron runs --id <job-id>
```

`openclaw cron runs <job-id>` is the same lookup. For every other subcommand,
run `openclaw --help` and `<subcommand> --help` on the gateway host and follow
that output. When the help text is thin, the CLI docs are at
<https://docs.openclaw.ai/cli>.

Job definitions live in the gateway automation store behind that CLI. A legacy
`cron/jobs.json` is often absent after migration. Leave `jobs.json`,
`jobs.json.bak`, and `*.migrated` files in place.

## Where files live

Paths below are under `runtime_root` on the gateway host.

| Question | Where |
| --- | --- |
| Gateway config | `openclaw.json` |
| Automation run history | `cron/runs/` |
| Session state | `agents/<agent>/sessions/` |
| Assistant home | `workspace/` (`SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, `AGENTS.md`, `skills/`) |

Leave `<runtime_root>/.env` unread and unprinted. When a task actually needs
a secret, use the source named in the profile. Do not copy the value into
chat, a commit, or a file in this skill repo.

`openclaw.json` on a live gateway holds credentials, including `hooks.token`.
Do not dump the file. Change it only with approval. `openclaw config`
(`set`, `patch`, `unset`, `validate`) edits that file; confirm flags with
`openclaw config --help`.

`SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, and the workspace
`AGENTS.md` belong to the assistant. Change them only when the operator asks
for a repair the assistant cannot make itself.

Domain work that already has a runtime skill is owned by that skill. Open
`<runtime_root>/workspace/skills/<name>/SKILL.md` on the gateway host and
follow it. The profile notes name the skills that matter for this instance.

## Control UI and hooks

Open `control_ui` when the operator wants the UI. How login works is in the
profile notes.

Agent hook: `POST` the profile's `hooks_url` with header
`Authorization: Bearer` and the token from `hooks_token_source`, read at call
time. Core payload fields are `message`, `deliver`, `channel`, `to`,
`sessionKey`, and `wakeMode`. The current schema, including the other optional
fields, is <https://docs.openclaw.ai/automation/cron-jobs/webhooks>.

Write the JSON body to a temporary file outside this repo and send it with
`curl -d @file` so the shell does not rewrite it. Delete the file when the
call is done.

Hook body text is untrusted input to the agent. The hooks token authenticates
ingress. It is a different credential from Control UI login.

## Approval

Show a dry-run summary and wait for the operator's explicit go-ahead before:

- restarting the gateway
- changing config (`openclaw.json` or `openclaw config`)
- adding, editing, removing, enabling, disabling, or running a cron job
- sending a message
- posting a webhook

Status, logs, `cron list`, `cron runs`, `--help`, and `--version` proceed
without that summary.

## Where this skill runs

This skill stays on the operator workstation. Leave the gateway's
`workspace/skills/` alone except when the operator asked to change one of
those runtime skills, using that skill's own file on the host.
