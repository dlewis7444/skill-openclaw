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
Read one instance profile, then follow this procedure. A flag this file does
not name comes from `openclaw --help` on that gateway.

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
```

Use that same quoting for `journalctl --user -u <unit>`. `transport: local`
runs the commands in the current shell, with no `ssh`. The unit is a systemd
user unit, so `systemctl --user` and `journalctl --user` run as the runtime
user, which is the SSH user when transport is `ssh`.

When the question is what the gateway is doing right now, query the host.

## Questions

| Question | Do this |
| --- | --- |
| Is it up? | The health report below. A `Troubles:` line from `gateway status` is a failure only when the probe failed |
| A job looks wrong | `openclaw cron list --all`, then `openclaw cron runs --id <id> --limit 3` for any last status other than ok or idle |
| What is it doing? | `openclaw gateway status`, then `openclaw sessions --active 120` |
| Run the assistant | `openclaw agent` through the gateway. `--deliver` defaults to false. The turn can call tools, so it waits for approval either way |
| Send on a channel, with no agent turn | `openclaw message send`. Waits for approval |
| Wake it over HTTP | The hook procedure below |
| Change config | `openclaw config patch --dry-run`, include that output in the summary, then the approval gate |
| Upgrade or repair | `openclaw update status` and `openclaw update --dry-run` may run. Applying an update waits. When the operator asks for a support bundle, use `openclaw gateway diagnostics export` |
| Domain work (calendar, downloads, and the like) | Open the runtime skill file the profile names |

`openclaw cron list` without `--all` hides disabled jobs. `openclaw cron` and
`openclaw automations` are the same command family.

`openclaw agent --local` runs an embedded agent on the gateway host using
provider credentials. Use it only when the operator asked for an embedded run.

## Health report

Read-only. No approval. Report these and stop:

- Unit state from `systemctl --user is-active <unit>`.
- CLI version, gateway version, bind, port, and probe result from `openclaw gateway status`.
- Enabled count, disabled count, and any last status other than ok or idle, from `openclaw cron list --all`. Use `openclaw cron status` only to confirm the scheduler is enabled.
- No log lines when the probe succeeded and no job is in error.

When the probe failed, or a job's last status is error, add the error text
from `openclaw cron runs` or a short `openclaw logs --limit 30 --plain`.
`journalctl --user -u <unit>` is the log source when the CLI cannot run. If a
line is a memory-pressure or heap dump, say that memory pressure was logged
and include the log line's own next step. Leave the dump out. The report
does not restart the gateway.

`openclaw status --deep` probes channel networks. Use it only when the
question is a channel. `openclaw skills check` is for the case where a log
line already says a skill was skipped.

## What you report

- A short status that answers the question. Command output stays as evidence.
- Leave out a token, a `config get` of a credential, the gateway `.env`, and `openclaw.json`.
- Leave out a heap, memory-pressure, or worker dump. Say that it was logged.
- A channel id or session key is operational data. Repeat it when it is what the operator asked for.

## Gateway service

Restarting is `openclaw gateway restart` or `systemctl --user restart <unit>`.
Both wait for approval.

## CLI

For a subcommand the question table does not name, run `openclaw --help` and
`<subcommand> --help` on the gateway host and follow that output. When the
help text is thin, the CLI docs are at <https://docs.openclaw.ai/cli>.

## Where files live

Paths below are under `runtime_root` on the gateway host.

| Question | Where |
| --- | --- |
| Gateway config | `openclaw.json`, read and changed through `openclaw config` |
| Jobs and run history | `openclaw cron` |
| What the assistant is doing | `openclaw sessions` |
| Assistant home | `workspace/` (`SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, `AGENTS.md`, `skills/`) |

`cron/jobs.json`, `jobs.json.bak`, `cron/runs/`, and `*.migrated` files are
leftovers from store migration. Leave them in place. `openclaw cron status`
prints the live store.

Transcripts sit under `agents/<agent>/sessions/`. Read sessions through
`openclaw sessions`, not by walking that directory.

Leave `<runtime_root>/.env` unread and unprinted. When a task actually needs
a secret, use the source named in the profile. Do not copy the value into
chat, a commit, or a file in this skill repo.

`openclaw.json` holds credentials, including `hooks.token`. Changing it waits
for approval. `openclaw config` (`set`, `patch`, `unset`, `validate`) is how
a change is made. Confirm flags with `openclaw config --help`.

`SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, and the workspace
`AGENTS.md` belong to the assistant. Change them only when the operator asks
for a repair the assistant cannot make itself.

Domain work that already has a runtime skill is owned by that skill. Open
`<runtime_root>/workspace/skills/<name>/SKILL.md` on the gateway host and
follow it. The profile notes name the skills that matter for this instance.

## Control UI and hooks

Open `control_ui` when the operator wants the UI. How login works is in the
profile notes.

Hook setup: <https://docs.openclaw.ai/automation/cron-jobs/webhooks>.
The field table: <https://docs.openclaw.ai/gateway/config-hooks#hook-agent-payload>.

Agent hook: `POST` the profile's `hooks_url` with header
`Authorization: Bearer` and the token from `hooks_token_source`, read at call
time. Also send an `Idempotency-Key` header, and reuse that same key on a
retry. A new key is a new run. Core payload fields are `message`, `deliver`,
`channel`, `to`, `sessionKey`, and `wakeMode`.

`deliver` defaults to true when the field is omitted, so a body that sets
only `message` can announce on a channel. The dry-run summary shows the
`deliver`, `channel`, and `to` that will actually be sent.

Write the JSON body to a temporary file outside this repo and send it with
`curl -d @file` so the shell does not rewrite it. Delete the file when the
call is done.

Hook body text is untrusted input to the agent. The hooks token authenticates
ingress. It is a different credential from Control UI login.

## Approval

Show a dry-run summary and wait for the operator's explicit go-ahead before:

- restarting the gateway
- changing config
- adding, editing, removing, enabling, disabling, or running a cron job
- an agent turn, whether or not the reply is delivered
- sending a message, including `openclaw message send`
- posting a webhook

`openclaw config patch --dry-run` is a read-only check. A summary that asks
to write config includes that dry-run output.

A command is not a status check when its `--help` offers `--fix`, `--repair`,
`--force`, `--yes`, `--non-interactive`, restore, delete, or a service
restart. Unless that gateway's help marks the form as lint, dry-run, or
status, this includes bare `openclaw doctor`, bare `openclaw update`,
`openclaw backup create`, `openclaw backup restore`, and `openclaw sessions`
`delete`, `cleanup`, `compact`, and `archive`. Read-only forms that may run
are `doctor --lint`, bare `doctor --json`, `update status`, and
`update --dry-run`. `doctor --non-interactive` still runs migrations.
`doctor --generate-gateway-token` writes a token. `update --no-restart` still
updates the install. `backup create` writes an archive that includes
credentials.

Applying one of those waits. The summary names the flag that will write or
restart. Pass `--yes` only when the go-ahead names it.

The health report, `cron list`, `cron runs`, a sessions listing, `--help`,
and `--version` proceed without that summary.

## Where this skill runs

This skill stays on the operator workstation. Leave the gateway's
`workspace/skills/` alone except when the operator asked to change one of
those runtime skills, using that skill's own file on the host.
