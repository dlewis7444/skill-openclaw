# Instance: example

Copy this file into the deployment's own project, outside this repo, and
point the skill's gitignored `.env` at it. `transport: local` drops `ssh` and
runs the CLI on this machine. Set `avoid_binary` only when a second
`openclaw` on `PATH` would shadow the right one.

```yaml
name: example
transport: ssh
ssh: oc@gateway.example.com
cli_prefix: PATH=/opt/openclaw/bin:$PATH
# avoid_binary: /usr/local/bin/openclaw
unit: openclaw-gateway.service
runtime_root: ~/.openclaw
control_ui: https://openclaw.example.com
hooks_url: https://openclaw.example.com/hooks/agent
hooks_token_source: pass example/openclaw/hooks-token
gateway_bind: lan
gateway_port: 18789
trusted_proxies:
  - 203.0.113.10
```

## Notes

Control UI auth, reverse-proxy headers, and the runtime skills to open go
here. Point at an architecture writeup when one exists. Keep tokens out of
this file.
