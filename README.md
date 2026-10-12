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
    uses: sv1nho/ci/.github/workflows/pipeline.yml@<commit-sha>
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
    uses: sv1nho/ci/.github/workflows/maintenance.yml@<commit-sha>
    with:
      e2e_base_url: http://127.0.0.1:3100
    secrets: inherit
```

Pin `@<commit-sha>` to a commit of this repo, the same way the workflows pin
their actions. The advisories job needs the repo secrets `AUTOMATION_APP_ID`
and `AUTOMATION_APP_KEY` (a GitHub App installed on the consumer repo); without
them only the nightly maintenance run fails, never the pipeline.
