# Contributing

## Architecture Overview

Berg is a fork of [HyperDX](https://hyperdx.io) / ClickStack, repurposed as a
web UI for AWS S3 Tables. It keeps HyperDX's log/discover UX but targets
analytical query workflows on S3 Tables instead of telemetry on ClickHouse.
It runs as three packages in a single monorepo:

| Package                 | Stack                                      | Role                                                                                             |
| ----------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| `packages/api`          | Express, Node 22+, Mongoose                | Auth, sessions, source / dashboard / saved-search CRUD, Athena query executor, Glue passthrough. |
| `packages/app`          | Next.js 16, Mantine, TanStack Query, Jotai | Search / Discover, dashboards, SQL editor, catalog browser.                                      |
| `packages/common-utils` | TypeScript                                 | Chart-config types, Lucene → Trino SQL emission, Athena type mapping, Zod schemas.               |

**Data flow at runtime:** browser → Next.js app → Express api → Athena (Trino)
reading S3 Tables via the Glue Data Catalog. MongoDB stores app metadata only
(users, teams, sources, dashboards, saved searches) — never your log data.

## Development

Pre-requisites:

- Docker (for MongoDB)
- Node.js (`>= 22`)
- Yarn (v4)
- AWS credentials with Athena + Glue read access (the api executes real
  queries against your account; see "AWS configuration" below)

Start a complete development stack:

```bash
yarn dev
```

This runs the Express api and the Next.js app locally and starts MongoDB in
Docker. Each git worktree gets its own slot-based port range so multiple
developers (or agents) can run `yarn dev` simultaneously without conflicts.
A dev portal at <http://localhost:9900> auto-starts and lists every active
stack with its assigned ports — check the portal to find the URL for your
instance.

Stop the stack:

```bash
yarn dev:down
```

The api and app are hot-reloaded — code edits are reflected immediately.

### AWS configuration

On first clone, copy the env templates and fill them in (the live filenames
are git-ignored to prevent accidentally committing real credentials):

```bash
cp packages/api/.env.development.example packages/api/.env.development
cp packages/app/.env.development.example packages/app/.env.development
cp packages/api/.env.test.example        packages/api/.env.test
cp packages/api/.env.e2e.example         packages/api/.env.e2e
cp packages/common-utils/.env.test.example packages/common-utils/.env.test
```

The api needs at least these to talk to AWS:

| Variable                  | Meaning                                                                              |
| ------------------------- | ------------------------------------------------------------------------------------ |
| `ATHENA_REGION`           | e.g. `us-east-1`.                                                                    |
| `ATHENA_WORKGROUP`        | Athena workgroup that issues the queries (default: `primary`).                       |
| `ATHENA_OUTPUT_LOCATION`  | `s3://<bucket>/` for Athena query results.                                           |
| `GLUE_CATALOG_ID`         | `<aws-account-id>:s3tablescatalog/<catalog-name>` for S3 Tables; unset for default.  |
| `GLUE_DATABASES`          | Comma-separated databases the UI should expose (`db1,db2`).                          |

Standard AWS credential resolution applies (`AWS_PROFILE`, IAM role, env vars).
The api fails fast at startup if Athena access is missing outside CI.

### Volumes

The dev stack mounts MongoDB data under `.volumes/` (per-worktree directory,
e.g. `.volumes/mongo_data_dev_42`). Delete the directory to reset app state.

### Windows / WSL 2

Hot module reload doesn't work out-of-the-box when the project lives on the
Windows side. Clone into a WSL home directory (`/home/<user>/...`, **not**
`/mnt/c/`) and run the dev commands from inside WSL. See
[the official WSL guide](https://code.visualstudio.com/docs/remote/wsl).

## Testing

All test environments use slot-based port isolation, so they can run in
parallel with the dev stack and across multiple worktrees.

### Unit Tests

Per-package — fastest feedback loop:

```bash
# from the package directory you want to test
yarn dev:unit          # watch mode
yarn ci:unit           # one-shot
```

Or across the whole repo:

```bash
make ci-unit
```

### Integration Tests

Spin up the full Docker dependencies and run a specific test file:

```bash
make dev-int-build                      # one-time build
make dev-int FILE=<TEST_FILE_NAME>      # Ctrl-C to tear down
```

### E2E Tests

Playwright against a local stack (api + app + MongoDB; AWS calls go through
fixture data via `.env.e2e`):

```bash
# first-time browser install
cd packages/app && yarn playwright install chromium

# run all
make e2e

# run a single spec, hot-reloaded
make dev-e2e FILE=navigation
make dev-e2e FILE=navigation GREP="help menu"
make dev-e2e GREP="should navigate"
make dev-e2e FILE=navigation REPORT=1   # open HTML report after run
make dev-e2e-clean                      # clear artifacts
```

E2E specs live in `packages/app/tests/e2e/`. Page objects under `page-objects/`,
shared components under `components/`.

### Lint + typecheck

```bash
make ci-lint
```

After finishing edits without a commit, run `yarn lint:fix` from the repo
root — pre-commit hooks already do this when you commit.

## AI-Assisted Development

The repo ships with `.claude/` agents and skills for test generation,
healing, and planning. They load automatically when you open the project in
Claude Code — no extra setup.

A Playwright MCP server config is included at `.cursor/mcp.json` for Cursor
users. Open **Cursor Settings → Tools & MCP** and enable the
`playwright-test` server to give Cursor's AI a live browser for test
exploration and debugging.

## Submitting changes

- Branch off `main`. For agent-generated branches, prefix with `claude/`,
  `agent/`, or `ai/` so the PR triage classifier can apply the right
  scrutiny.
- Keep PRs scoped to a single logical change; explain the **why** in the
  description, not just the what.
- Run `make ci-lint` and `make ci-unit` before opening the PR.
- Use the git author's default profile in commits — no `Co-Authored-By`
  trailers.

If you need help, open a GitHub issue with a minimal reproduction.
