# Issue tracker: GitHub (fork workflow)

This repo is a **fork**, not the canonical repo. Issues, specs, and PRs all live on the upstream repo — the fork itself is a push target for branches, not where issues or PRs get filed.

- **`origin`** → `xsmJohnnyDepp/oreilly-ingest` (your fork — push feature branches here)
- **`upstream`** → `mosaibah/oreilly-downloader` (canonical repo — issues, specs, and PR destination live here)

Use the `gh` CLI for all operations, always targeting `upstream` explicitly with `--repo mosaibah/oreilly-downloader` (or `-R` shorthand) since `gh` would otherwise infer the repo from `origin`, which is wrong here.

## Conventions

- **Create an issue**: `gh issue create --repo mosaibah/oreilly-downloader --title "..." --body "..."`. Use a heredoc for multi-line bodies.
- **Read an issue**: `gh issue view <number> --repo mosaibah/oreilly-downloader --comments`, filtering comments by `jq` and also fetching labels.
- **List issues**: `gh issue list --repo mosaibah/oreilly-downloader --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'` with appropriate `--label` and `--state` filters.
- **Comment on an issue**: `gh issue comment <number> --repo mosaibah/oreilly-downloader --body "..."`
- **Apply / remove labels**: `gh issue edit <number> --repo mosaibah/oreilly-downloader --add-label "..."` / `--remove-label "..."` (only if you have write access; as an external contributor you likely don't — leave labeling to the maintainer)
- **Close**: usually not available to external contributors; leave open with a final comment instead

## Before starting new work

Sync local `main` with upstream before branching, since `origin` (the fork) can drift behind:

```bash
git fetch upstream
git merge upstream/main    # or: git rebase upstream/main
```

Do feature work on a branch, push it to `origin` (your fork) — never to `upstream` directly, you won't have write access:

```bash
git push -u origin <branch-name>
```

## Opening a pull request

PRs go from your fork's branch to `upstream`'s `main`:

```bash
gh pr create --repo mosaibah/oreilly-downloader \
  --head xsmJohnnyDepp:<branch-name> \
  --base main \
  --title "..." --body "..."
```

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag. Not relevant here since you're the one opening PRs, not triaging others'.)_

## When a skill says "publish to the issue tracker"

Create a GitHub issue on `mosaibah/oreilly-downloader` (upstream), not on your fork.

## When a skill says "fetch the relevant ticket"

Run `gh issue view <number> --repo mosaibah/oreilly-downloader --comments`.

## Wayfinding operations

Used by `/wayfinder`. All map/child issues below live on **upstream** (`mosaibah/oreilly-downloader`), so add `--repo mosaibah/oreilly-downloader` to every `gh issue` / `gh api` call in this section. The **map** is a single issue with **child** issues as tickets.

- **Map**: a single issue labelled `wayfinder:map`, holding the Notes / Decisions-so-far / Fog body. `gh issue create --repo mosaibah/oreilly-downloader --label wayfinder:map`.
- **Child ticket**: an issue linked to the map as a GitHub sub-issue (`gh api` on the sub-issues endpoint). Where sub-issues aren't enabled, add the child to a task list in the map body and put `Part of #<map>` at the top of the child body. Labels: `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, the ticket is assigned to the driving dev.
- **Blocking**: GitHub's **native issue dependencies** — the canonical, UI-visible representation. Add an edge with `gh api --method POST repos/mosaibah/oreilly-downloader/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>`, where `<blocker-db-id>` is the blocker's numeric **database id** (`gh api repos/mosaibah/oreilly-downloader/issues/<n> --jq .id`, _not_ the `#number` or `node_id`). GitHub reports `issue_dependencies_summary.blocked_by` (open blockers only — the live gate). Where dependencies aren't available, fall back to a `Blocked by: #<n>, #<n>` line at the top of the child body. A ticket is unblocked when every blocker is closed.
- **Frontier query**: list the map's open children (`gh issue list --repo mosaibah/oreilly-downloader --state open`, scoped to the map's sub-issues / task list), drop any with an open blocker (`issue_dependencies_summary.blocked_by > 0`, or an open issue in the `Blocked by` line) or an assignee; first in map order wins.
- **Claim**: `gh issue edit <n> --repo mosaibah/oreilly-downloader --add-assignee @me` — the session's first write. Requires write access; if you don't have it, note the claim in a comment instead.
- **Resolve**: `gh issue comment <n> --repo mosaibah/oreilly-downloader --body "<answer>"`, then close via the maintainer if you lack close permission, then append a context pointer (gist + link) to the map's Decisions-so-far.
