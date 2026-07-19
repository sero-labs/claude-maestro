# Configure GPT‑5.6 (+ Claude) via CLIProxyAPI

Run the **Claude Code** CLI and all its tooling, but backed by **OpenAI Codex (GPT‑5.6)** through a
localhost‑only [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) gateway — while still being able to
switch to the real Anthropic Claude models in the same session.

- The proxy also serves **`gpt-5.6-terra`**, **`gpt-5.6-luna`**, and the genuine **Claude** models.
- Your normal `claude` and `codex` commands are left completely untouched.

Tested on **macOS (Apple Silicon), zsh, Homebrew**. Notes for Linux/WSL are included where they differ.

---

## What you need

- `claude` (Claude Code CLI) — https://docs.claude.com/en/docs/claude-code
- An **OpenAI account with Codex / GPT‑5.6 access** (for the Codex OAuth login)
- An **Anthropic / claude.ai account** (optional — only if you want the real Claude models too)
- Homebrew (macOS) or a package manager / release binary (Linux)

---

## 1. Install CLIProxyAPI

**macOS (Homebrew):**
```bash
brew install cliproxyapi
```
This installs the `cliproxyapi` binary and a `brew services` (launchd) unit. Its default config path is
`$(brew --prefix)/etc/cliproxyapi.conf` (e.g. `/opt/homebrew/etc/cliproxyapi.conf`).

**Arch Linux:** `yay -S cli-proxy-api-bin` (ships an example config + a systemd **user** service)
**Other Linux:** use the official installer at `router-for-me/cliproxyapi-installer` (download, inspect, then run).
**Windows:** grab the current binary from the project's GitHub Releases.

Confirm it installed and exposes the OAuth flags:
```bash
cliproxyapi --version
cliproxyapi --help | grep -E 'codex-login|claude-login|config'
```

---

## 2. Write the config

Create the config directory and file. This binds to **localhost only**, disables TLS and remote
management, and uses round‑robin routing.

```bash
mkdir -p ~/.cli-proxy-api
chmod 700 ~/.cli-proxy-api

# Generate a strong random local API key and store a client copy (never hard-code it)
openssl rand -hex 32 > ~/.cli-proxy-api/client.key
chmod 600 ~/.cli-proxy-api/client.key
```

Write `~/.cli-proxy-api/config.yaml` (substitute the key from `client.key` into `api-keys`):

```yaml
host: "127.0.0.1"
port: 8317
tls:
  enable: false
  cert: ""
  key: ""
remote-management:
  allow-remote: false
  secret-key: ""
  disable-control-panel: true
auth-dir: "~/.cli-proxy-api"
api-keys:
  - "PASTE_THE_CONTENTS_OF_client.key_HERE"
debug: false
logging-to-file: false
usage-statistics-enabled: false
request-retry: 3
max-retry-credentials: 0
max-retry-interval: 30
routing:
  strategy: "round-robin"
  session-affinity: false
ws-auth: true
```

One-liner to inject the key so the file and `client.key` always match:
```bash
KEY=$(cat ~/.cli-proxy-api/client.key)
# then edit config.yaml so the api-keys entry equals "$KEY"
chmod 600 ~/.cli-proxy-api/config.yaml
```

> **Never** leave the shipped placeholder keys (`your-api-key-1`, `sk-dummy`, …). CLIProxyAPI disables
> endpoints if an unsafe placeholder remains.

**Homebrew note:** `brew services` reads `$(brew --prefix)/etc/cliproxyapi.conf`. Point it at your user
config with a symlink (back up the original first):
```bash
BREW_CONF="$(brew --prefix)/etc/cliproxyapi.conf"
[ -f "$BREW_CONF" ] && [ ! -L "$BREW_CONF" ] && cp -p "$BREW_CONF" "$BREW_CONF.orig.bak"
ln -sf ~/.cli-proxy-api/config.yaml "$BREW_CONF"
```

---

## 3. Authenticate the upstream account(s)

Run the OAuth flows with your config. Each opens a browser (use `--no-browser` to print a URL to open
manually); approve in the browser and it completes on `localhost`.

**Codex / GPT‑5.6 (required):**
```bash
cliproxyapi -config ~/.cli-proxy-api/config.yaml -codex-login
```

**Claude / Anthropic (optional — enables the real Claude models):**
```bash
cliproxyapi -config ~/.cli-proxy-api/config.yaml -claude-login
```

Lock down the credential files each login writes:
```bash
chmod 600 ~/.cli-proxy-api/*.json
```

---

## 4. Start the service

**macOS:**
```bash
brew services start cliproxyapi     # or: brew services restart cliproxyapi
```

**Linux (systemd user service):**
```bash
systemctl --user daemon-reload
systemctl --user enable --now cliproxyapi   # unit name per your package
```

Confirm it is up and **listening on 127.0.0.1 only**:
```bash
lsof -nP -iTCP:8317 -sTCP:LISTEN         # macOS
# ss -ltnp | grep 8317                    # Linux
```

## 5. Verify

```bash
KEY="$(<"$HOME/.cli-proxy-api/client.key")"

# Unauthenticated -> must be rejected (401)
curl -s -o /dev/null -w 'unauth: %{http_code}\n' http://127.0.0.1:8317/v1/models

# Authenticated -> capture the status code AND the body, so a non-200 is caught
CODE=$(curl -s -w '%{http_code}' -o /tmp/cpa_models.json \
  http://127.0.0.1:8317/v1/models -H "Authorization: Bearer $KEY")
echo "auth: $CODE"
grep -oE '"(gpt-5\.6[^"]*|claude-[^"]*)"' /tmp/cpa_models.json | sort -u

# Generation actually works? (a 200 model list does NOT prove the upstream login
# is valid — this catches a stale Codex/Claude token that /v1/models misses)
ANTHROPIC_BASE_URL=http://127.0.0.1:8317 ANTHROPIC_AUTH_TOKEN="$KEY" \
CLAUDE_CODE_ALWAYS_ENABLE_EFFORT=1 ENABLE_TOOL_SEARCH=false \
claude -p "reply with the single word: ok" --model gpt-5.6-sol --effort low \
  --bare --max-turns 1 --output-format json \
  | grep -q '"is_error":false' && echo "generation: OK" || echo "generation: FAILED — re-run the provider login"
```

Expected: unauthenticated `401`; authenticated `200` listing `gpt-5.6-sol` (plus
`terra`, `luna`); `generation: OK`. If the model list is `200` but generation
`FAILED`, the proxy is up but the upstream login is stale — re-run the relevant
`-codex-login` / `-claude-login` (see the troubleshooting note below).

---

## 6. Uninstall / remove

> **Read before running.** These steps stop the proxy and delete your local
> config **and the OAuth credential files** (`~/.cli-proxy-api/*.json`) — those
> are live access tokens for your OpenAI / Anthropic / Google accounts. Removing
> them logs this machine out of the proxy. It does **not** revoke the grant
> upstream: to fully kill access, also revoke the app in your provider's account
> settings (OpenAI / Google security pages). There is no undo.

**1. Stop the service.**
```bash
brew services stop cliproxyapi                 # macOS
# systemctl --user disable --now cliproxyapi    # Linux (unit name per your package)
```

**2. Restore the Homebrew config symlink** (macOS — undoes the symlink step 2 made):
```bash
BREW_CONF="$(brew --prefix)/etc/cliproxyapi.conf"
[ -L "$BREW_CONF" ] && rm -f "$BREW_CONF"
[ -f "$BREW_CONF.orig.bak" ] && mv "$BREW_CONF.orig.bak" "$BREW_CONF"
```

**3. Remove Maestro's state** — config, key, OAuth credentials, and the
overrides file:
```bash
rm -rf ~/.cli-proxy-api
```

**4. (Optional) Remove the binary.** `cliproxyapi` is a machine-shared tool you
may route other things through — only uninstall it if nothing else uses it:
```bash
brew uninstall cliproxyapi                      # macOS
```

After this, Claude Maestro's preflight will fail with connection-refused until
you re-run the setup — that's expected.

---

## Notes & troubleshooting

- **Localhost only.** The proxy binds `127.0.0.1`. Never expose port 8317 beyond your machine.
- **Connection refused:** the service isn't running — `brew services start cliproxyapi` (or the systemd
  equivalent), then re-check the `lsof`/`ss` listener.
- **401 when authenticated:** the `api-keys` value in `config.yaml` doesn't match `client.key`. Fix and
  `brew services restart cliproxyapi`.
- **A model is missing from `/v1/models`:** the matching OAuth account isn't logged in / lacks access.
  Re-run the relevant `-codex-login` / `-claude-login`.
- **Keep secrets secret.** Don't paste or commit `client.key`, `config.yaml`, or the `*.json` OAuth files.
  This guide contains none of them — the config above uses a placeholder you replace locally.