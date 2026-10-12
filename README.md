# ci

Shared, reusable GitHub Actions workflows for the projects that share the same
stack (React + Vite, Express + Prisma + Postgres, Better Auth). A change made
here reaches every consumer, so the CI does not drift between projects.

## What's here

- `.github/workflows/pipeline.yml` — the push/PR pipeline: quality (lint, types,
  knip), build + first-load budget + a production-install boot, the test suites
  behind a schema-drift check, the browser journeys, a CodeQL scan, and a
  dependency + secret scan.
- `.github/workflows/maintenance.yml` — the nightly pass: form and API fuzzing,
  and an advisories job that opens a single pull request as a GitHub App.

Both are `workflow_call` libraries. They assume the consumer repo exposes the
usual npm scripts (`lint`, `typecheck`, `knip`, `build`, `size`, `db:check`,
`test`, `fuzz`, `e2e`, `advisories`) and a `package-lock.json`. CI-internal
values (database name, boot port) are standardized here because each run is
isolated; only the browser-test base URL differs per project and is an input.

## Using them

In a consumer repo, `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
permissions:
  contents: read
  security-events: write
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  ci:
    uses: sv1nho/ci/.github/workflows/pipeline.yml@v1
    with:
      e2e_base_url: http://127.0.0.1:3100
    secrets: inherit
```

And `.github/workflows/maintenance.yml`:

```yaml
name: Maintenance
on:
  schedule:
    - cron: '0 3 * * *'
  workflow_dispatch:
permissions:
  contents: write
  pull-requests: write
jobs:
  maintenance:
    uses: sv1nho/ci/.github/workflows/maintenance.yml@v1
    with:
      e2e_base_url: http://127.0.0.1:3100
    secrets: inherit
```

Consumers pin `@v1`, a moving tag on this repo. A backward-compatible change
here is released by moving `v1` to the new commit (`git tag -f v1 <sha> && git
push -f origin v1`); every consumer picks it up on its next run, with no edit
on their side. A breaking change gets a new tag (`v2`) that consumers adopt
deliberately. The third-party actions inside these workflows stay pinned by
commit SHA, so moving `v1` never pulls in unreviewed third-party code.

The advisories job needs the repo secrets `AUTOMATION_APP_ID` and
`AUTOMATION_APP_KEY` (a GitHub App installed on the consumer repo); without
them only the nightly maintenance run fails, never the pipeline.
