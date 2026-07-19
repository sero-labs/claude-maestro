---
description: First-run setup for Claude Maestro — install & configure the CLIProxyAPI proxy, then verify it serves models
---

Optional focus (e.g. a specific provider to log in, or "just verify"): $ARGUMENTS

Set up the local CLIProxyAPI proxy that Claude Maestro routes through. The
authoritative steps live in `{baseDir}/references/setup-cli-proxy-api.md`
— **read that file first** and treat it as the source of truth; the notes below
are only how to drive it.

**How to run this**
- Work step by step and make it idempotent: before each step, check whether it's
  already done and skip it if so. A re-run on a working install should end at
  "already good — nothing to do".
- Run the safe, non-interactive steps yourself (install check, config scaffolding,
  key generation, starting the service, the verify curl).
- **You cannot do the OAuth logins** — they open a browser and require the user.
  When you reach them, STOP and tell the user to run the exact command themselves
  by typing it with a leading `!` in the prompt (e.g.
  `! cliproxyapi -config ~/.cli-proxy-api/config.yaml -codex-login`), so the
  output lands in this session. Wait for them before continuing.
- Never print, echo, or commit the generated key, `config.yaml`, or the `*.json`
  OAuth files. When you need the key, read the file at use time.

**The path, per the reference guide**
1. **Install** the `cliproxyapi` binary (Homebrew on macOS; the platform note for
   Linux/Windows). Confirm with `cliproxyapi --version`.
2. **Config + key** — create `~/.cli-proxy-api/` (perms `700`), generate
   `client.key` with `openssl rand -hex 32` (perms `600`), and write
   `config.yaml` (localhost-only, TLS off, remote-management off) with its
   `api-keys` entry set to the key's contents. On macOS, symlink the brew config
   at it. Skip anything already present. Also drop a starter
   `~/.cli-proxy-api/maestro.env` with the default overrides commented out
   (see the skill's Config section) so the user has one place to set their
   preferred model/effort later — don't overwrite it if it already exists.
3. **Authenticate** the upstream account(s) — hand these to the user as `!`
   commands: `-codex-login` (required for the gpt-5.6 models), plus
   `-claude-login`, `-gemini-login`, etc. for whatever else they want. If
   `$ARGUMENTS` names a provider, do just that one.
4. **Start** the service (`brew services start cliproxyapi`, or the systemd user
   unit on Linux) and confirm it's listening on `127.0.0.1:8317` only.
5. **Verify** — run Claude Maestro's preflight: an unauthenticated
   `/v1/models` must return `401`, and an authenticated call must return `200`
   with `gpt-5.6-sol` (plus any other logged-in models) in the list. Read the key
   from `~/.cli-proxy-api/client.key` for the authenticated call.

**Finish** by telling the user, in plain English, which models are now available
and that they can start with e.g. "ask sol …" or `/maestro:consult`. If a step fails,
point at the matching entry in the reference guide's troubleshooting section
rather than improvising.

**To undo this later:** the reference guide's "Uninstall / remove" section stops
the proxy and scrubs the local config and OAuth credentials. If the user asks to
remove or uninstall Maestro, follow those steps — read the warning aloud
first, confirm before deleting anything, and leave the shared `cliproxyapi`
binary unless they explicitly want it gone.
