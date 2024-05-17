# Neon PR Preview Action

This is a single action designed to manage the lifecycle of Neon database branches for PR Preview environments.

## Examples

### Fly.io - NodeJS

```yaml
name: PR Review
on:
  # Run this workflow on every PR event. Existing review apps will be updated when the PR is updated.
  # Neon branches are created and removed according to PR updates
  pull_request:
    types: [opened, reopened, synchronize, closed]

jobs:
  pr-preview:
    runs-on: ubuntu-latest
    # Only run one deployment at a time per PR.
    concurrency:
      group: pr-${{ github.event.number }}

    environment:
      name: review
      url: ${{ steps.deploy.outputs.url }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4

      - run: npm install

      - uses: neondatabase/pr-review-branch@v1

      - run: npm run db:migrate

      - id: deploy
        uses: superfly/fly-pr-review-apps@1.2.0
        with:
          secrets: DATABASE_URL=${{ env.DATABASE_URL }}
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```


## Inputs

2 inputs are required by the action, all others have defaults. Inputs can be passed as either environment variables or parameters to the action; parameters take precidence over environment variables.

### Required Inputs

The action requires the following inputs:

- `NEON_PROJECT_ID`
  - `https://console.neon.tech/app/projects/NEON_PROJECT_ID`
  - Can be found at the end of the console URL
- `NEON_API_KEY`
  - https://console.neon.tech/app/settings/api-keys
  - Can be created in Neon console account settings

### Input Types

All inputs can be passed as either environment variables or parameters to the action. Parameters to the action will override any environment variables available to the action. Therefore if you pass different values as an environment variable and parameter the action will use the parameter.

Here an example of parameters overriding environment variables:

```yaml
on:
  pull_request:
    types: [opened, reopened, synchronize, closed]

env:
  NEON_DATABASE_NAME: that_base

jobs:
  pr-preview:
    runs-on: ubuntu-latest
    steps:
      # exports DATABASE_URL to use in next steps
      - uses: neondatabase/pr-review-branch@v1
        with:
          database_name: no_treble
```

The `pr-review-branch` action will use the value `no_treble` despite the fact that `that_base` was set to the `NEON_DATABASE_NAME` environment variable. If we didn't pass the `database_name` parameter the action would have used the environment variable.

## Inputs

| Environment Variable   | Parameter         | Required | Default                                 | Description                                                        |
| ---------------------- | ----------------- | -------- | --------------------------------------- | ------------------------------------------------------------------ |
| `NEON_PROJECT_ID`      | `project_id`      | ✅       | 🚫                                      |
| `NEON_API_KEY`         | `api_key`         | ✅       | 🚫                                      |
| `NEON_BRANCH_NAME`     | `branch_name`     |          | `preview/pr-${{ github.event.number }}` |
| `NEON_DB_NAME`         | `db_name`         |          | `neondb`                                |
| `NEON_ROLE_NAME`       | `role_name`       |          | `neondb_owner`                          |
| `NEON_PARENT_NAME`     | `parent_name`     |          | Primary Branch                          |
| `NEON_SUSPEND_TIMEOUT` | `suspend_timeout` |          | 0                                       |
| `NEON_SSL_MODE`        | `ssl_mode`        |          | 'require'                               | Supported values are `require`, `verify-ca`, `verify-full`, `omit` |
| `NEON_ORM`             | `orm`             |          | 'none'                                  | Will attempt to detect via code inspection. Supported values are `none`, `drizzle`, `prisma`                   |

## Outputs

The action exports two kinds of outputs. The first output type is available to the step outputs (link to github actions help) and the second output type is commonly used environment variables.

### Output Variables

These values are exported into the actions output values.

| Output               | Description                                              |
| -------------------- | -------------------------------------------------------- |
| `db_url`             | `postgres://` database connection URL                    |
| `db_url_with_pooler` | `postgres://` database connection pooler URL (pgbouncer) |
| `host_name`          | `postgres://...@host_name/`                              |
| `host_with_pooler`   | `postgres://...@host_with_pooler/`                       |
| `branch_id`          | Neon Branch ID                                           |
| `password`           | Database password                                        |
| `role_name`          | Database role name (default value unless specified)      |
| `database_name`      | Database name (default value unless specified)           |
| `parent_name`        | Parent branch name used (default value unless specified) |

To access these output variables you should set an `id` for the action so you can reference that in later steps. The following example uses the step id `create-branch` which makes the output variables in the above table available as `${{ steps.create-branch.outputs.$X }}`

```yaml
on:
  pull_request:
    # only create branches on open, without closed this will not delete branches
    types: [opened]
jobs:
  pr-preview:
    runs-on: ubuntu-latest
    steps:
      # exports DATABASE_URL to use in next steps
      - uses: neondatabase/pr-review-branch@v1
        id: create-branch
        with:
          project_id: rapid-haze-373089
          api_key: ${{ secrets.NEON_API_KEY }}
      - run: echo db_url ${{ steps.create-branch.outputs.db_url }}
      - run: echo host ${{ steps.create-branch.outputs.host }}
      - run: echo branch_id ${{ steps.create-branch.outputs.branch_id }}
```

### Exported Environment Variables

The GitHub Action exports the following environment variables. Environment variables can be access in later steps via `${{ env.DATABASE_URL }}` or in shell runs as `$DATABASE_URL`.

| Exported Environment Variables | Description                                                  |
| ------------------------------ | ------------------------------------------------------------ |
| `DATABASE_URL`                 | Database URL, may change based on `orm` value.               |
| `DIRECT_URL`                   | Non Connection Pooler URL. Only exported when `orm='prisma'` |


## Events

This action needs to listen for the following major PR events:

- opened
  - creates a new branch
- reopened
  - creates a new branch or resets an existing branch
- synchronize
  - creates a new branch or resets an existing branch
- closed
  - deletes the branch

These events should be listed in the `pull_request` event `types` at the top of the action.

```yaml
on:
  pull_request:
    types: [opened, reopened, synchronize, closed]
```
