# ci

A reusable GitHub Actions workflow that skips a check workflow's push run on `main` when the merged
PR already passed it on the same files.

A repo runs its checks on every pull request and on every push to `main`. After a squash merge,
`skip-if-tested.yml` outputs `tested=true` only when all of these hold:

1. The pushed commit belongs to a PR merged into the pushed branch.
2. `main` before the push is an ancestor of the PR head, so the PR branch was up to date.
3. The PR head has the same tree (the same files) as the pushed commit.
4. Every PR from the head's branch targets the pushed branch, so no run merged the head into
   another one.
5. A `pull_request` run of the same workflow file, from the PR's head branch and repo, passed on
   the PR head after the last change of any such PR's base.

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
    uses: gpahal/ci/.github/workflows/skip-if-tested.yml@ee5b876c99dc4af15f288630704e48421a637716 # v1.1.0
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
- Protect `main` against force-pushes. A PR run tests the head merged into `main` as it was then,
  and only a `main` that moves forward keeps that state in the head.
- On pull requests the gate always outputs `false`, so required checks are unaffected. The PR
  shows the gate as `gate / gate`.

## License

[MIT](LICENSE)

