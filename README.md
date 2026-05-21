# stackmerge-action

Merge an entire stack of pull requests with one label. Each branch is
rebased onto the trunk after its predecessor merges, then auto-merges
when CI passes.

Works with any stacked-PR workflow — [git-spice], [Graphite], or
manually stacked branches. The action doesn't depend on either tool and
never touches their navigation comments.

[git-spice]: https://abhinav.github.io/git-spice/
[Graphite]: https://graphite.dev/

## How it works

1. You add a label (default: `merge-stack`) to the top-most PR of the
   stack you want to ship.
2. The **start** action walks down the base chain of that PR, applies
   the same label to every PR below it, and enables GitHub's native
   squash-auto-merge on the bottom PR (the one whose base is the trunk).
3. GitHub merges the bottom PR when its CI passes.
4. The **continue** action fires on each merge: it rebases every
   remaining labeled branch onto its new base in turn (using
   `git rebase --onto <new-base> <pre-merge-tip>` to cleanly drop the
   squash-merged ancestor commits), force-pushes the rebased SHAs, and
   enables auto-merge on the new bottom PR.
5. Repeat until the entire chain is merged.

If any branch fails to rebase cleanly, the continue action strips the
label from every remaining labeled PR and comments on each with the
failed run URL — so the stack pauses safely instead of leaving
half-rebased branches around.

### Labeling a non-top PR

You can label **any** PR in the stack — the action merges everything
up to and including the labeled PR. For example, in a stack of
`#57 → #58 → #59 → #60 → #61 → #62`, labeling **#59** merges #57,
#58, and #59 in order, then stops.

PRs that sit above the labeled PR in the same stack (here: #60, #61,
#62) are **rebased** onto the new `main` so they don't show stale
"conflicting" status in the GitHub UI, but they are **not merged**.
If you decide to ship them later, label the top of the remaining
stack and the cascade resumes.

## Prerequisites

In the repository where you're using this action:

- **Settings → General → Pull Requests → "Allow auto-merge"** must be
  enabled.
- **Settings → General → Pull Requests → "Automatically delete head
  branches"** is strongly recommended. GitHub auto-retargets dependent
  PRs to the trunk when the merged branch is deleted, which this action
  relies on.
- CI should run **only on PRs targeting your trunk branch**, e.g.:
  ```yaml
  on:
    pull_request:
      branches: [main]
  ```
  Otherwise CI runs redundantly on every intermediate PR. (This is also
  the standard pattern for stacked-PR repos in general.)

A token secret in your repo or org:

- A **Personal Access Token (classic)** with `repo` scope, or a
  fine-grained PAT with `Contents: write` and `Pull requests: write`.
- The built-in `GITHUB_TOKEN` will **not** work; see
  [Why a PAT?](#why-a-pat-and-not-github_token) below.

## Usage

Add two thin caller workflows to your repository.

### `.github/workflows/merge-stack-start.yml`

```yaml
name: Merge Stack — Start
on:
  pull_request:
    types: [labeled]
permissions:
  contents: read
  pull-requests: write
concurrency:
  group: merge-stack-start-${{ github.event.pull_request.number }}
  cancel-in-progress: false
jobs:
  start:
    # Only act on human-applied `merge-stack` labels on non-fork PRs.
    # The sender check prevents recursive firing when the action labels
    # the next PR down the chain itself.
    if: >-
      github.event.label.name == 'merge-stack'
      && github.event.sender.type == 'User'
      && github.event.pull_request.head.repo.fork == false
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    steps:
      - uses: kalbasit/stackmerge-action/start@main
        with:
          token: ${{ secrets.GHA_PAT_TOKEN }}
```

### `.github/workflows/merge-stack-continue.yml`

```yaml
name: Merge Stack — Continue
on:
  pull_request:
    types: [closed]
permissions:
  contents: write
  pull-requests: write
concurrency:
  group: merge-stack-continue-${{ github.repository }}
  cancel-in-progress: false
jobs:
  continue:
    if: >-
      github.event.pull_request.merged == true
      && contains(github.event.pull_request.labels.*.name, 'merge-stack')
      && github.event.pull_request.head.repo.fork == false
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: kalbasit/stackmerge-action/continue@main
        with:
          token: ${{ secrets.GHA_PAT_TOKEN }}
```

Replace `GHA_PAT_TOKEN` with whatever you've named your PAT secret.
Pin to a tag (e.g. `@v1`) instead of `@main` once you've validated the
behavior in your repo.

## Inputs

Both actions accept the same inputs:

| Input   | Default       | Description                                    |
|---------|---------------|------------------------------------------------|
| `token` | _(required)_  | GitHub token. PAT with `repo` scope.           |
| `label` | `merge-stack` | The label that triggers stack merging.         |
| `trunk` | `main`        | The trunk branch name.                         |

If you customize `label`, remember to update the `if:` condition in
your caller workflows to match.

## Permissions

Your caller workflow must grant at least:

- **start**: `contents: read`, `pull-requests: write`
- **continue**: `contents: write`, `pull-requests: write`

## Why a PAT and not `GITHUB_TOKEN`?

GitHub's built-in `GITHUB_TOKEN` has two limitations that block this
workflow:

1. It cannot enable auto-merge on PRs it didn't author.
2. Commits pushed with `GITHUB_TOKEN` do **not** trigger other
   workflows. Since the continue action force-pushes restacked
   branches, those branches need to re-run CI for the next auto-merge
   to fire — that requires a PAT.

## Caveats

- The action only operates on PRs from the same repository — fork PRs
  are skipped. Stacks pretty much only work intra-repo anyway.
- Triggers use `pull_request` (not `pull_request_target`); the calling
  workflow file is fetched from the PR head, so changes to the caller
  take effect on the same PR that introduces them. This matches how
  most CI workflows are configured.
- The "find the bottom of the chain" logic explicitly excludes the
  just-merged PR by number and head-ref to work around GitHub's
  eventual consistency on `--state open`.
- The action does not regenerate any tool-specific navigation comments
  (e.g. git-spice's stack tree). The user's local view stays the
  source of truth for those.

## License

Apache 2.0 — see [LICENSE](LICENSE).
