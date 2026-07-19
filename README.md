# Claude Maestro

**A second opinion from any model, without leaving Claude Code.**

Route a question, a code review, or a whole task to GPT, Gemini, Grok — whatever
you're logged into — from inside your Claude Code session. Your session stays on
Claude; only the delegated call goes to the other model. New models show up
automatically as you log into them.

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code)
- [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) running locally, with
  at least one non-Anthropic account logged in (e.g. OpenAI/Codex for the GPT
  models). `/maestro:setup` walks you through this — you don't need to set it up
  by hand first.

## Install

```
/plugin marketplace add sero-labs/claude-maestro
/plugin install maestro@sero-labs
/reload-plugins
```

## Setup

Maestro reaches your models through a small local proxy. First time only:

```
/maestro:setup
```

It installs and configures the proxy, helps you log into your model provider(s),
and verifies everything works. You run the browser logins yourself; it handles the
rest.

## Use it

Just ask — no slash command needed:

- *"ask sol if this migration is safe — src/db/migrate.ts"*
- *"get gpt's take on this design"*
- *"have terra review my changes"*

Or use the shortcuts:

| Command | What it does |
|---|---|
| `/maestro:consult <question>` | A second opinion (read-only) |
| `/maestro:critique <paths>` | Ranked issues, each with a concrete fix |
| `/maestro:implement <task>` | Hand a model a task to build — you spec it and verify |
| `/maestro:setup` | First-run proxy setup |

Name a model ("ask terra…") or let it use the default. Fuzzy names are fine — it
matches against whatever your proxy is actually serving, and tells you which model
it picked before it calls.

## Make your own flows

Beyond consult / critique / implement, you can mint your own reusable flows:

```
/maestro:new-flow "review a GitHub PR by number"
```

That scaffolds a new flow and its own `/maestro:<name>` command. Two examples ship
in the box:

- `/maestro:code-review` — review a diff, branch, or set of files
- `/maestro:pr-review` — review a GitHub PR by number or URL

## When something breaks

Routed calls depend on your provider login staying fresh. If a call hangs or
errors, it's almost always a stale login — re-run it (e.g.
`cliproxyapi -config ~/.cli-proxy-api/config.yaml -codex-login`) and retry. The
**Troubleshooting** section in `SKILL.md` has a symptom → fix table, where to find
the real error in the proxy logs, and a quick way to tell an infrastructure
problem from a prompt problem.

## How it works (and one important note)

Your Claude Code session runs on your normal Anthropic login. Maestro **never**
routes that login through the proxy — it sets the routing only on the individual
delegated calls to other providers. (Routing an Anthropic subscription login
through a proxy would break Anthropic's terms; this doesn't.)

It's also built to be easy on your context: it passes file *paths* to the other
model rather than pasting in file contents, and caps what comes back — so a second
opinion doesn't eat half your window.

## Credits

Inspired by [claude-roundtable](https://github.com/shujaurrehmanbaloch/claude-roundtable/)
and [this post from Theo](https://x.com/theo/status/2075757587010929003).

## License

[MIT](LICENSE)
