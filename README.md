# pr-dependency-guard

A GitHub Action that blocks merging a PR until its declared prerequisite PRs are merged.

It reads dependencies from two places:

- **PR body keywords** — `Depends on #123`, `Blocked by #45`, `After #7`
- **PR labels** — any label matching `depends-on-<N>` (e.g. `depends-on-123`)

When any referenced PR is not yet merged (or does not exist), the action fails its check, which — once you set it as a Required Status Check in your branch protection rules — blocks the merge button.

A sticky comment on the PR summarizes the current state and updates as the situation changes.

## Quick start

Add `.github/workflows/pr-dependency-check.yml` to your repo:

```yaml
name: PR Dependency Check
on:
  pull_request:
    types: [opened, edited, synchronize, reopened, labeled, unlabeled]

permissions:
  pull-requests: write
  contents: read

jobs:
  check-dependencies:
    runs-on: ubuntu-latest
    steps:
      - uses: BoBeenLee/pr-dependency-guard@v1
        with:
          target-branches: 'main,develop'
```

Then go to **Settings → Branches → Branch protection rules** for `main` (and `develop`), edit the rule, and add `check-dependencies` (the job name) as a **Required status check**.

## Declaring a dependency

In the PR body:

```markdown
This PR adds the consumer side of the new event payload.

Depends on #42
Blocked by #51
```

Or with labels — create labels named `depends-on-42`, `depends-on-51` on the repo and apply them to the PR. Both sources are unioned together.

Both patterns are case-insensitive. HTML comments (`<!-- ... -->`) inside the PR body are stripped before matching, so commented-out examples in PR templates do not trigger the check.

## Inputs

| Input | Default | Description |
|---|---|---|
| `github-token` | `${{ github.token }}` | Token with `pull-requests: write` |
| `target-branches` | `main,develop,master` | Comma-separated. PRs targeting any other branch pass silently. |
| `bypass-label` | `skip-dependency-check` | When this label is on the PR, the check passes and the sticky comment records the bypass. |
| `keywords` | `Depends on,Blocked by,After` | Comma-separated keywords recognized in the PR body before `#N`. |
| `label-prefix` | `depends-on-` | Label format. `depends-on-123` → PR #123. |

## Bypass

Apply the `skip-dependency-check` label (configurable) to a PR to skip enforcement for emergency hotfixes or other exceptional situations. The sticky comment will note that the bypass was used, leaving an audit trail. Remove the label to re-enable the check.

## Limitations

- **Dependency PR getting merged does not auto-retrigger the downstream PR's check.** The check re-runs only when the downstream PR itself fires an event (commit pushed, label changed, body edited, etc.). To force a re-check after a prerequisite is merged, edit the PR description (no-op edit is fine) or push an empty commit.
- **Same-repo references only.** Cross-repo references like `owner/repo#N` are not supported in this version.
- **Single-level dependencies.** Transitive dependencies are not resolved — if PR A depends on PR B, and PR B depends on PR C, PR A's check only verifies B's state, not C's.

## License

MIT
