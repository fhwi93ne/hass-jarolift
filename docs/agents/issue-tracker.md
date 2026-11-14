# Issue tracker: GitHub

Issues and specs for this repo live as GitHub issues. Use the `gh` CLI for all operations.

## Conventions

- **Create an issue**: `gh issue create --title "..." --body "..."`. Use a heredoc for multi-line bodies.
- **Read an issue**: `gh issue view <number> --comments`, filtering comments by `jq` and also fetching labels.
- **List issues**: `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'` with appropriate `--label` and `--state` filters.
- **Comment on an issue**: `gh issue comment <number> --body "..."`
- **Apply / remove labels**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **Close**: `gh issue close <number> --comment "..."`

**This repo is a fork** (upstream: `4cc8783b/hass-jarolift`). Inside a fork clone, `gh issue`/`gh pr` default to the upstream repo, so **always pass `--repo fhwi93ne/hass-jarolift`** to every `gh issue`, `gh pr` and `gh label` command (or run `gh repo set-default fhwi93ne/hass-jarolift` once per clone). For `gh api` calls use `repos/fhwi93ne/hass-jarolift/...`. Never file issues on the upstream repo.

## REST fallback (when `gh issue` / `gh pr` fail)

In some environments (e.g. Claude cloud sessions) GitHub GraphQL is blocked, so `gh issue …`, `gh pr …` and `gh label list` fail with `HTTP 403: GitHub GraphQL is not available`. When that happens, use these `gh api` (REST) equivalents instead. `R=repos/fhwi93ne/hass-jarolift`.

- **Create**: `gh api $R/issues -f title="..." -F body=@body.md -f 'labels[]=needs-triage'` (write multi-line bodies to a file first; repeat `-f 'labels[]=...'` per label). Prints the new issue; `--jq .number` for just the number.
- **Read**: `gh api $R/issues/<n> --jq '{number, title, state, body, labels: [.labels[].name]}'` plus comments via `gh api $R/issues/<n>/comments --jq '[.[] | {user: .user.login, body}]'`
- **List**: `gh api "$R/issues?state=open&labels=<label>&per_page=100" --jq '[.[] | select(.pull_request | not) | {number, title, body, labels: [.labels[].name]}]'`. The REST endpoint also returns PRs, hence the `select`. Comma-separate multiple labels (AND).
- **Comment**: `gh api $R/issues/<n>/comments -F body=@comment.md`
- **Add label**: `gh api $R/issues/<n>/labels -f 'labels[]=<label>'` (creates the label if missing). **Remove label**: `gh api -X DELETE $R/issues/<n>/labels/<label>` (URL-encode `:` as `%3A`).
- **Create label explicitly**: `gh api $R/labels -f name="<label>" -f color="ededed"`
- **Close**: comment first, then `gh api -X PATCH $R/issues/<n> -f state=closed -f state_reason=completed` (or `not_planned`).
- **Assign to self** (wayfinder claim): `gh api $R/issues/<n>/assignees -f 'assignees[]=fhwi93ne'`
- **Sub-issue** (wayfinder child): `gh api $R/issues/<map>/sub_issues -F sub_issue_id=<child-db-id>`, where the db id is `gh api $R/issues/<child> --jq .id`.
- **PR read / diff**: `gh api $R/pulls/<n>` and `gh api $R/pulls/<n> -H "Accept: application/vnd.github.diff"`; list via `gh api "$R/pulls?state=open"`.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag.)_

When set to `yes`, PRs run through the same labels and states as issues, using the `gh pr` equivalents:

- **Read a PR**: `gh pr view <number> --comments` and `gh pr diff <number>` for the diff.
- **List external PRs for triage**: `gh pr list --state open --json number,title,body,labels,author,authorAssociation,comments` then keep only `authorAssociation` of `CONTRIBUTOR`, `FIRST_TIME_CONTRIBUTOR`, or `NONE` (drop `OWNER`/`MEMBER`/`COLLABORATOR`).
- **Comment / label / close**: `gh pr comment`, `gh pr edit --add-label`/`--remove-label`, `gh pr close`.

GitHub shares one number space across issues and PRs, so a bare `#42` may be either: resolve with `gh pr view 42` and fall back to `gh issue view 42`.

## When a skill says "publish to the issue tracker"

Create a GitHub issue.

## When a skill says "fetch the relevant ticket"

Run `gh issue view <number> --comments`.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single issue with **child** issues as tickets.

- **Map**: a single issue labelled `wayfinder:map`, holding the Notes / Decisions-so-far / Fog body. `gh issue create --label wayfinder:map`.
- **Child ticket**: an issue linked to the map as a GitHub sub-issue (`gh api` on the sub-issues endpoint). Where sub-issues aren't enabled, add the child to a task list in the map body and put `Part of #<map>` at the top of the child body. Labels: `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, the ticket is assigned to the driving dev.
- **Blocking**: GitHub's **native issue dependencies**, the canonical, UI-visible representation. Add an edge with `gh api --method POST repos/fhwi93ne/hass-jarolift/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>`, where `<blocker-db-id>` is the blocker's numeric **database id** (`gh api repos/fhwi93ne/hass-jarolift/issues/<n> --jq .id`, _not_ the `#number` or `node_id`). GitHub reports `issue_dependencies_summary.blocked_by` (open blockers only, the live gate). Where dependencies aren't available, fall back to a `Blocked by: #<n>, #<n>` line at the top of the child body. A ticket is unblocked when every blocker is closed.
- **Frontier query**: list the map's open children (`gh issue list --state open`, scoped to the map's sub-issues / task list), drop any with an open blocker (`issue_dependencies_summary.blocked_by > 0`, or an open issue in the `Blocked by` line) or an assignee; first in map order wins.
- **Claim**: `gh issue edit <n> --add-assignee @me`, the session's first write.
- **Resolve**: `gh issue comment <n> --body "<answer>"`, then `gh issue close <n>`, then append a context pointer (gist + link) to the map's Decisions-so-far.
