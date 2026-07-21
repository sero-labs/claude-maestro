---
name: maestro
description: Bring routed frontier models into the Claude Code harness for consult, critique, or implementation via the local CLIProxyAPI proxy — currently nicknamed sol, terra, and luna (fetch /v1/models if the list may have changed; more providers like Gemini/Grok/Kimi show up automatically once logged in). No slash command required — trigger whenever the user asks to route work to a non-Anthropic model, e.g. "ask terra", "with sol", "use luna", "get a second opinion from gpt", or names any model that appears in the proxy's /v1/models list. Typos are fine; match against the live model list. If a name is ambiguous (could be a person, tool, or file rather than a model), confirm before routing rather than assuming. /maestro:consult, /maestro:critique, /maestro:implement exist as explicit shortcuts but are not required.
---

# Claude Maestro — consult / critique / implement on routed models

You are the ORCHESTRATOR, running on this session's normal Anthropic login.
NEVER set `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` on this session's own
environment — routed env vars go inline on individual headless calls only.
(Anthropic's terms ban routing a Claude subscription login through a proxy;
this flow never does that.)

For Claude-to-Claude mixing (Fable ↔ Sonnet ↔ Opus) no routing is needed:
the user can `/model` mid-session. This skill is for the OTHER subscriptions.

## Config

| Env var | Default | Meaning |
|---|---|---|
| `MODEL_ROUTER_URL` | `http://127.0.0.1:8317` | proxy address |
| `MODEL_ROUTER_KEY_FILE` | `$HOME/.cli-proxy-api/client.key` | file holding the proxy API key — read at call time, never sent as-is |
| `MODEL_ROUTER_MODEL` | first non-Claude model served (resolved live at preflight; `gpt-5.6-sol` only if the list can't be read) | default routed model |
| `MODEL_ROUTER_EFFORT` | `high` | fallback effort when the task doesn't imply one |

Effort is chosen per call — see **EFFORT** below. `MODEL_ROUTER_EFFORT` is only
the last-resort fallback.

**Overrides file (optional).** Set machine-wide defaults in one place instead of
exporting vars by hand: put any of the above in `~/.cli-proxy-api/maestro.env`.
Every command block below sources it if present, so its values flow into the
`${VAR:-default}` expansions; each key is optional and a missing one falls back
to the default in the table. A per-call choice (the user names a model, or you
resolve `<effort>`) still wins over the file.

```bash
# ~/.cli-proxy-api/maestro.env — every line optional
export MODEL_ROUTER_MODEL=gpt-5.6-terra   # preferred default model
export MODEL_ROUTER_EFFORT=high           # IMPLEMENT fallback effort
export MODEL_ROUTER_URL=http://127.0.0.1:8317
# export MODEL_ROUTER_KEY_FILE=...        # only if the key lives off the standard path
```

**Model picking (fuzzy, forgiving):** the user speaks casually — "with terra",
"ask sol", "use luna" — and may typo ("dera", "tera"). Map whatever they said
to the closest ID in the proxy's `/v1/models` list (e.g. "terra" →
`gpt-5.6-terra`). Fetch the list if you haven't this session. Always state
which model you picked ("Routing to gpt-5.6-terra") BEFORE the call, so a
wrong guess is visible immediately. If nothing plausibly matches, use the
default and say so.

**Resolution order when the user names no model** (highest wins): a **flow's
declared `model`** (only when you're running a custom flow that sets one) →
`MODEL_ROUTER_MODEL` (if the user set it) → the **session default resolved at
preflight** (the first non-Claude model the proxy actually serves) → the static
`gpt-5.6-sol` last resort if that lookup failed. Use the resolved value without
comment. This mirrors how `effort` resolves — a caller who *does* name a model
still overrides everything, including a flow's `model`. Because the default is
resolved from the live `/v1/models` list, no specific model nickname is assumed
to exist on any given machine.

## EFFORT — dynamic, chosen per call

Effort controls how hard the routed model thinks (and how long you wait). Set
it per call by reading the task in front of you — you are the judge; don't
keyword-match. Valid values: `low`, `medium`, `high`, `xhigh`, `max` (if unsure
a value is supported, use `high`).

- **Read the request and match the cognitive load.** A quick lookup, a
  mechanical edit, or a "does this smell right?" glance → `low`/`medium`. A
  subtle design, architecture, or security judgement, or a spec the worker must
  reason through → `high`/`xhigh`. Reserve `max` for when the payoff clearly
  justifies the latency.
- **The caller overrides you.** If the user or a slash command names an effort —
  literally (`--effort low`) or in plain words ("quick take", "think hard about
  this", "deep dive") — honor that over your own read.
- **Fallbacks when nothing points either way:** CONSULT/CRITIQUE → `high` (the
  remote model carries the thinking); IMPLEMENT → `MODEL_ROUTER_EFFORT` (the
  spec carries the thinking).
- **State it with the model, before the call** — "Routing to gpt-5.6-terra at
  `high` effort." A wrong pick is then visible immediately, same as the model.

In the commands below, `<effort>` is the value you resolved here.

## Preflight — (RUN ON SKILL LOAD) once per session, before the first routed call

TWO checks, because they fail for different reasons. `/v1/models` can return 200
while the actual upstream login (Codex, etc.) is stale — so a models-list check
alone gives false confidence and you find out only after a multi-minute call dies.
The second call proves generation really works, in ~2 seconds.

```bash
[ -f "$HOME/.cli-proxy-api/maestro.env" ] && . "$HOME/.cli-proxy-api/maestro.env"
URL="${MODEL_ROUTER_URL:-http://127.0.0.1:8317}"
KEY="$(cat "${MODEL_ROUTER_KEY_FILE:-$HOME/.cli-proxy-api/client.key}")"

# 1) proxy reachable + serving models
curl -s "$URL/v1/models" -H "Authorization: Bearer $KEY" | head -c 400; echo

# session default model: when none is configured, pick the first NON-Claude model
# the proxy actually serves (Claude models are excluded — routing a Claude login
# through the proxy is off-limits). gpt-5.6-sol is only a last resort if the list
# can't be read. Announce it so the resolved default is visible.
: "${MODEL_ROUTER_MODEL:=$(curl -s "$URL/v1/models" -H "Authorization: Bearer $KEY" | grep -oE '"id"[[:space:]]*:[[:space:]]*"[^"]*"' | cut -d'"' -f4 | grep -vi claude | head -1)}"; : "${MODEL_ROUTER_MODEL:=gpt-5.6-sol}"
echo "SESSION DEFAULT MODEL: $MODEL_ROUTER_MODEL"

# 2) generation actually works — catches a stale upstream login that check 1 misses
ANTHROPIC_BASE_URL="$URL" ANTHROPIC_AUTH_TOKEN="$KEY" \
CLAUDE_CODE_ALWAYS_ENABLE_EFFORT=1 ENABLE_TOOL_SEARCH=false \
claude -p "reply with the single word: ok" \
  --model "$MODEL_ROUTER_MODEL" --effort low \
  --bare --max-turns 1 --output-format json \
  | grep -q '"is_error":false' && echo "PREFLIGHT OK" || echo "PREFLIGHT FAILED"
```

Read the outcome:
- **Connection refused** (check 1) → proxy isn't running. Start it
  (`brew services start cliproxyapi`, or the Linux systemd user unit), re-check.
- **Check 1 OK but check 2 says FAILED** (`auth_unavailable` / `Service
  Unavailable` / a 5xx in the body) → proxy is up but the upstream login for that
  model is **stale or expired**. This is the common failure. STOP and tell the
  user to re-run that provider's login, e.g.
  `cliproxyapi -config ~/.cli-proxy-api/config.yaml -codex-login` (swap
  `-codex-login` for `-claude-login`, `-gemini-login`, etc. per the model).
- **Empty model list / auth error on check 1** → same re-login fix.

If this fails on a fresh machine, run `/maestro:setup` (walks the whole
install), or see `{baseDir}/references/setup-cli-proxy-api.md` for the detailed
setup guide.

## TROUBLESHOOTING

Routed calls go through an upstream you don't control, so plan for the off day.
This is the fast path from symptom to fix.

**First move — where the real error lives.** The client only ever sees a generic
`500`. The proxy logs the *actual* upstream cause to `~/.cli-proxy-api/logs/`. On
any failure, read the newest message log — one grep usually names it outright:

```bash
ls -t ~/.cli-proxy-api/logs/error-v1-messages-*.log | head -1 \
  | xargs grep -hoE '"message":"[^"]*"' | sort -u
```

(e.g. `auth_unavailable: no auth available (providers=codex, ...)` = a stale
Codex login — the single most common failure.)

**Symptom → cause → fix:**

| What you see | Likely cause | Fix |
|---|---|---|
| Hangs for minutes, then `500` / `Service Unavailable` / `auth_unavailable` | upstream login (Codex etc.) **stale/expired** — even though `/v1/models` still returns `200` | re-run that provider's login: `cliproxyapi -config ~/.cli-proxy-api/config.yaml -codex-login` (or `-claude-login`, …), then retry |
| `curl /v1/models` → connection refused | proxy not running | `brew services start cliproxyapi` (or the Linux systemd user unit) |
| Looks frozen, no output for minutes | `--output-format json` prints nothing until done, and high effort is genuinely slow — not a hang | wait, lower the effort, or background the call; confirm infra with rung 1 below |
| Wall time huge but the model barely ran | proxy silently retried a failing upstream (`request-retry` in config.yaml) | it's the stale-login case above — re-login instead of waiting out retries |
| `JSONDecodeError: Invalid control character` parsing the reply | the reply contains raw newlines | write the JSON to a file, read `result` with a tolerant parser — not a one-line inline pipe |
| One model 5xx's but others answer | only that provider's login is down | route to a Claude model on the same proxy as a fallback |

**Bisect infra vs. your task — a 3-rung ladder**, cheapest first. Where it first
breaks tells you what's wrong (swap `gpt-5.6-sol` below for any model your proxy
serves — see the `SESSION DEFAULT MODEL` line from preflight):

```bash
# rung 1 — pure routing, ~2s. Fails → login/proxy problem, NOT your task.
claude -p "reply with the single word: ok" --model gpt-5.6-sol --effort low --bare --max-turns 1 --output-format json
# rung 2 — add a tool. Fails only here → tool/permission issue.
claude -p "run \`git status --short\` and report the file count" --model gpt-5.6-sol --effort low --bare --max-turns 4
# rung 3 — your real call. Fails only here → it's the prompt / effort / size, not infra.
```

**Conduct.** Don't blindly retry — a stale login won't heal itself, so retry
**once**, then STOP and surface the re-login fix; each retry is another slow wait
for nothing. Before a call you expect to be slow, tell the user (~"2-3 min") so
silence reads as *working*, not *dead*.

## CONSULT FLOW — a second opinion (read-only)

```bash
[ -f "$HOME/.cli-proxy-api/maestro.env" ] && . "$HOME/.cli-proxy-api/maestro.env"
: "${MODEL_ROUTER_MODEL:=$(curl -s "${MODEL_ROUTER_URL:-http://127.0.0.1:8317}/v1/models" -H "Authorization: Bearer $(cat "${MODEL_ROUTER_KEY_FILE:-$HOME/.cli-proxy-api/client.key}" 2>/dev/null)" 2>/dev/null | grep -oE '"id"[[:space:]]*:[[:space:]]*"[^"]*"' | cut -d'"' -f4 | grep -vi claude | head -1)}"; : "${MODEL_ROUTER_MODEL:=gpt-5.6-sol}"
ANTHROPIC_BASE_URL="${MODEL_ROUTER_URL:-http://127.0.0.1:8317}" \
ANTHROPIC_AUTH_TOKEN="$(cat "${MODEL_ROUTER_KEY_FILE:-$HOME/.cli-proxy-api/client.key}")" \
CLAUDE_CODE_ALWAYS_ENABLE_EFFORT=1 CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=3 ENABLE_TOOL_SEARCH=false \
claude -p "<question, with relevant file/dir PATHS for it to read>" \
  --model "$MODEL_ROUTER_MODEL" --effort <effort> \
  --bare --max-turns 15 --output-format json \
  --disallowedTools "Edit,Write,NotebookEdit"
```

Prompt rules:
- Pass PATHS, never file contents — the worker reads them on its own quota.
- End every consult prompt with: "Return your verdict and top points in
  30 lines or fewer."
- The JSON output includes a `session_id` — keep it for follow-ups.

**Worked example.** User: *"ask terra if this migration is safe — src/db/migrate.ts"*
1. Announce: "Routing to `gpt-5.6-terra` at `high` effort" (schema/data-loss
   judgement → high).
2. Run the CONSULT block with `--model gpt-5.6-terra`, `--effort high`, and
   prompt `"Is the migration in src/db/migrate.ts safe to run against a
   populated prod table? Read the file. Return your verdict and top points in
   30 lines or fewer."`
3. Relay terra's verdict inline; stash the `session_id` in case the user wants
   a follow-up.

## CRITIQUE FLOW — same call as CONSULT, different framing

Prompt shape: "Critique <paths or described change>: strengths, top 5 issues
ranked by severity, one concrete fix each. 40 lines or fewer."

## FOLLOW-UP — cross-model memory (the important trick)

The remote model remembers its own session. To continue a consult/critique
(e.g. hand it Fable's counterpoint), resume — send ONLY the new delta:

```bash
[ -f "$HOME/.cli-proxy-api/maestro.env" ] && . "$HOME/.cli-proxy-api/maestro.env"
: "${MODEL_ROUTER_MODEL:=$(curl -s "${MODEL_ROUTER_URL:-http://127.0.0.1:8317}/v1/models" -H "Authorization: Bearer $(cat "${MODEL_ROUTER_KEY_FILE:-$HOME/.cli-proxy-api/client.key}" 2>/dev/null)" 2>/dev/null | grep -oE '"id"[[:space:]]*:[[:space:]]*"[^"]*"' | cut -d'"' -f4 | grep -vi claude | head -1)}"; : "${MODEL_ROUTER_MODEL:=gpt-5.6-sol}"
ANTHROPIC_BASE_URL="${MODEL_ROUTER_URL:-http://127.0.0.1:8317}" \
ANTHROPIC_AUTH_TOKEN="$(cat "${MODEL_ROUTER_KEY_FILE:-$HOME/.cli-proxy-api/client.key}")" \
CLAUDE_CODE_ALWAYS_ENABLE_EFFORT=1 CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=3 ENABLE_TOOL_SEARCH=false \
claude -p --resume <session_id> "<only the new information or counterpoint>" \
  --model "$MODEL_ROUTER_MODEL" --effort <effort> \
  --bare --max-turns 15 --output-format json \
  --disallowedTools "Edit,Write,NotebookEdit"
```

Never re-paste what the remote model already saw.

## IMPLEMENT FLOW — build via a routed model (one compound call per task)

```bash
[ -f "$HOME/.cli-proxy-api/maestro.env" ] && . "$HOME/.cli-proxy-api/maestro.env"
: "${MODEL_ROUTER_MODEL:=$(curl -s "${MODEL_ROUTER_URL:-http://127.0.0.1:8317}/v1/models" -H "Authorization: Bearer $(cat "${MODEL_ROUTER_KEY_FILE:-$HOME/.cli-proxy-api/client.key}" 2>/dev/null)" 2>/dev/null | grep -oE '"id"[[:space:]]*:[[:space:]]*"[^"]*"' | cut -d'"' -f4 | grep -vi claude | head -1)}"; : "${MODEL_ROUTER_MODEL:=gpt-5.6-sol}"
ANTHROPIC_BASE_URL="${MODEL_ROUTER_URL:-http://127.0.0.1:8317}" \
ANTHROPIC_AUTH_TOKEN="$(cat "${MODEL_ROUTER_KEY_FILE:-$HOME/.cli-proxy-api/client.key}")" \
CLAUDE_CODE_ALWAYS_ENABLE_EFFORT=1 CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=3 ENABLE_TOOL_SEARCH=false \
claude -p "<self-contained task spec>" \
  --model "$MODEL_ROUTER_MODEL" \
  --effort <effort — default ${MODEL_ROUTER_EFFORT:-high}> \
  --bare --max-turns 30 --permission-mode acceptEdits \
&& ls -la <expected output paths> \
&& grep -c "<acceptance marker>" <built file>
```

The spec must be self-contained: exact file paths, what to create or change,
acceptance criteria, what NOT to touch, and "verify your work, then report
changed paths + how you verified". You make every design call in the spec —
the worker just types. Misses go back as a fix delegation with the failure
evidence attached; you do not fix code yourself.

## CUSTOM FLOWS — user-defined variants (loaded on demand)

consult / critique / implement are the built-in primitives. Anything more
specific — a code review, a PR review off a GitHub number, a test-writer, a
changelog drafter — lives as a named flow in `{baseDir}/flows/<name>.md`, read
only when it's actually needed so this file stays lean.

**Running a custom flow.** When the user's request matches a flow in the index
below (by name, or by its "when to use"), read that flow file. Each flow is a
short spec: a `mode`, a default `effort`, an optional preferred `model`, how to
normalize its inputs, and the prompt shape to send. It does NOT repeat the bash —
you run it through the matching delegate call already defined above:

- `mode: read-only` → the CONSULT / CRITIQUE block (Edit/Write disallowed). Bash
  stays available, so a flow can still run `gh`, `git diff`, etc. on its own quota.
- `mode: write` → the IMPLEMENT block (`--permission-mode acceptEdits` + the
  `&&`-verify tail).

The flow supplies the *what* (prompt + effort + inputs + an optional preferred
`model`); the *how* (env vars, `--resume`, the TOKEN RULES below) is unchanged.
If the flow declares a `model`, substitute it literally for the `--model` value
(exactly as you do for a caller-named model) — unless the caller named one, which
still wins. If the flow omits `model`, leave `--model "$MODEL_ROUTER_MODEL"`
as-is so the resolved session default applies. Announce the resolved model +
effort before the call exactly as always.

**Available flows:**

<!-- MAESTRO:FLOWS — managed by /maestro:new-flow; keep both markers, one bullet per flow -->
- **code-review** — review a diff, branch, range, or files; ranked issues + fixes, no edits → `flows/code-review.md`
- **pr-review** — review a GitHub PR by number/URL; delegate fetches it with `gh` → `flows/pr-review.md`
<!-- /MAESTRO:FLOWS -->

**Adding a flow.** Run `/maestro:new-flow "<description>"` — or just ask (e.g.
"make a maestro flow that reviews a GitHub PR"). It scaffolds the flow file, a
slash-command wrapper, and the index line above. The flow-file shape lives in
`{baseDir}/flows/_TEMPLATE.md`.

## TOKEN RULES (non-negotiable — this is why the skill exists)

1. Paths, never contents, in delegate prompts.
2. Cap every response ("≤30 lines") — whatever comes back lives in this
   session's context forever.
3. Keep independent opinions independent — chain Fable→Sol only when one must
   critique the other's output. To overlap your own in-session analysis with a
   routed call, background it (`claude -p ... &`) and `wait`; otherwise the
   Bash call blocks and there's no real parallelism.
4. Follow-ups go through `--resume`, sending only the delta.
5. Never read a whole built file back into context — verify with targeted
   `grep -c` / `head` / `tail` slices.