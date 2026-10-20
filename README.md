# ci

A reusable GitHub Actions workflow that skips a check workflow's push run on `main` when the merged
PR already passed it on the same files.

A repo runs its checks on every pull request and on every push to `main`. After a squash merge,
`skip-if-tested.yml` outputs `tested=true` only when all of these hold:

1. The pushed commit belongs to a merged PR.
2. `main` before the push is an ancestor of the PR head, so the PR branch was up to date.
3. The PR head has the same tree (the same files) as the pushed commit.
4. A run of the same workflow file passed on the PR head.

Otherwise, including on any API error, it outputs `tested=false` and the checks run. When
`main` moved after a PR's run, the merge lands files no run tested, so the checks run.

## Usage

```yaml
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  gate:
    uses: gpahal/ci/.github/workflows/skip-if-tested.yml@<commit-sha> # v1.0.0
    permissions:
      actions: read
      contents: read
      pull-requests: read

  test:
    needs: gate
    if: needs.gate.outputs.tested != 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "the repo's own steps"
```

Give each job you want skipped `needs: gate` and that `if`. Pin the tag's commit SHA, with the
tag in a comment.

- The `permissions` block is required. A called workflow gets only the permissions its caller job
  grants.
- A job that needs a skipped job is skipped too. Give such a job
  `if: ${{ !cancelled() && !failure() }}`.
- The whole PR run must pass. If any job failed, even one that isn't a required check, the push
  run runs everything.
- On pull requests the gate always outputs `false`, so required checks are unaffected. The PR
  shows the gate as `gate / gate`.

## License

[MIT](LICENSE)

