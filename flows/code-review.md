---
name: code-review
mode: read-only
effort: high
inputs: file paths, a git ref/range (e.g. `main..HEAD`), or nothing (defaults to the current working diff)
requires: git
---

# Maestro flow: code-review

**When to use:** the user wants a routed model to review code — a diff, a branch, a commit range, or a set of files — and report issues rather than change anything.

**Normalize the input:** if `$ARGUMENTS` names paths, review those files. If it names a git ref/range, the delegate runs `git diff <range>` itself. If it's empty, default to the current working diff (`git diff` + `git diff --staged`). Pass the paths/range only — never paste diff contents into the delegate prompt.

**Prompt shape:** "Review <target>. Read the files and run `git diff <range>` yourself as needed. Report, in this order: (1) a one-line overall verdict — ship / needs work / block; (2) top issues ranked by severity, each with `file:line` and one concrete fix; (3) any merge blockers or missing tests. Ranked, most severe first. 40 lines or fewer."

**After it returns:** relay the review inline. Add your own take only where you'd reprioritize or disagree, then stop. Keep the `session_id` in case the user wants to push back on a finding via `--resume`.
