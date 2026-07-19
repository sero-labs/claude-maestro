---
name: pr-review
mode: read-only
effort: high
inputs: a GitHub PR number or URL
requires: gh (GitHub CLI, authenticated)
---

# Maestro flow: pr-review

**When to use:** the user wants a routed model to review a GitHub pull request identified by number or URL.

**Normalize the input:** accept `#482`, `482`, `owner/repo#482`, or a full `github.com/owner/repo/pull/482` URL. **Keep the repo context if it's there** — pass the full URL, or the number together with `--repo owner/repo`. Only fall back to a bare number when the user is plainly working in that repo's checkout, because `gh pr view 482` with no repo resolves against the current directory and can inspect the wrong repository. Either way pass just the reference, never the diff — the delegate fetches it itself, so the PR contents never enter this session.

**Prompt shape:** "Review GitHub PR <ref>. Fetch it yourself with `gh pr view <ref>` (title, description, checks) and `gh pr diff <ref>` (the changes) — include `--repo owner/repo` if you were given one. Report, in this order: (1) one line on what the PR intends; (2) an overall verdict — approve / request changes / block; (3) top issues ranked by severity, each with `file:line` and one concrete fix; (4) any merge blockers, missing tests, or scope creep. Ranked, most severe first. 40 lines or fewer."

**After it returns:** relay the review inline. Add your own take only where you'd reprioritize, then stop. Keep the `session_id` so the user can hand the delegate a maintainer's counterpoint via `--resume` without re-fetching the PR.
