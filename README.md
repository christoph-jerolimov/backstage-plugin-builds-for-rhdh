# backstage-plugin-builds-for-rhdh

GitHub Actions workflows that package Backstage plugins as dynamic plugins for
[Red Hat Developer Hub (RHDH)](https://developers.redhat.com/products/developer-hub/overview)
and push the resulting OCI images to [Quay.io](https://quay.io).

The workflows clone an external plugin repository, build a workspace, package it
with the [`@red-hat-developer-hub/cli`](https://www.npmjs.com/package/@red-hat-developer-hub/cli)
`plugin package` command, and publish the image to
`quay.io/christophs-backstage-plugin-builds-for-rhdh/<imageName>:<imageTag>`.

## How it works

There is one reusable build workflow and a set of trigger workflows per source
repository.

### Reusable build (`build.yaml`)

The core workflow. It can be started manually (`workflow_dispatch`) or called by
another workflow (`workflow_call`). It:

1. Hardens the runner and installs `podman` and Node.js 22.
2. Clones `repo`, checks out `branch` (shallow).
3. In `<workdir>` runs `yarn install --immutable`, `yarn tsc:full`,
   and `yarn build:all`.
4. Packages the plugin via `npx @red-hat-developer-hub/cli@1.10.x plugin package
   --tag "<imageName>:<imageTag>"` and verifies the build log contains the
   expected success message.
5. Stamps a `quay.expires-after=2w` label on the image so Quay.io auto-deletes
   it two weeks after the push.
6. Logs in to Quay.io and pushes the image.

| Input       | Required | Description                                  |
| ----------- | -------- | -------------------------------------------- |
| `repo`      | yes      | Git repository URL to clone                  |
| `branch`    | yes      | Branch or ref to checkout                    |
| `workdir`   | yes      | Working directory under the cloned repository |
| `imageName` | yes      | Name of the image to build                   |
| `imageTag`  | no       | Image tag (defaults to `latest`)             |

The build job only runs when triggered by the actor `christoph-jerolimov`.

Required secrets (passed through from the trigger workflows): `QUAY_USERNAME`,
`QUAY_TOKEN`.

### Trigger workflows

For each source repository there are three trigger workflows:

- **`*-main.yaml`** — manual trigger that builds a single `workspace` from the
  repository's `main` branch. Validates that `workspace` matches `^[a-zA-Z-]+$`.
- **`*-pr.yaml`** — manual trigger that builds a single `workspace` from a pull
  request. Validates `workspace` and that `pr` matches `^[0-9]+$`, then builds
  `refs/pull/<pr>/head` and tags the image `pr-<pr>`.
- **`*-cron.yaml`** — scheduled daily build (also manually dispatchable) that
  builds a fixed matrix of workspaces.

### Source repositories

| Prefix              | Repository                                              | Cron schedule          | Image name prefix |
| ------------------- | ------------------------------------------------------ | ---------------------- | ----------------- |
| `backstage-*`       | `backstage/backstage`                                  | `0 2 * * 0` (Sun 02:00) | `backstage-`      |
| `bcp-*`             | `backstage/community-plugins`                          | `0 3 * * *` (daily 03:00) | `bcp-`         |
| `rhdh-plugins-*`    | `redhat-developer/rhdh-plugins`                        | `0 4 * * *` (daily 04:00) | `rhdh-`        |
| `proberaum-*`       | `proberaum/backstage-plugins`                          | `0 5 * * *` (daily 05:00) | `proberaum-`   |

Cron schedules use standard [cron syntax](https://en.wikipedia.org/wiki/Cron)
(`minute hour day-of-month month day-of-week`, in UTC). The daily schedules are
staggered an hour apart so the builds do not all run at once; `backstage` runs
weekly because of its large matrix.

Cron matrices:

- **backstage:** catalog, home
- **community-plugins:** acr, argocd, bookmarks, jfrog-artifactory,
  multi-source-security-viewer, nexus-repository-manager, npm, ocm, quay, rbac,
  servicenow, tekton, topology
- **rhdh-plugins:** adoption-insights, ai-integrations, app-defaults,
  bulk-import, extensions, global-header, homepage, lightspeed, quickstart,
  scorecard, theme, translations
- **proberaum:** analytics-viewer, config-viewer, devtools, env-viewer,
  github-notifications, hcloud, icon-viewer, permissions-viewer,
  scheduler-notifications, whats-new

The `backstage/backstage` repository keeps its plugins under `plugins/` rather
than `workspaces/`, so its trigger workflows pass `plugins/<workspace>` as the
build `workdir`.

## Usage

To build a plugin on demand, run the matching `*-main` or `*-pr` workflow from
the GitHub Actions tab and provide the `workspace` (and `pr` for PR builds).
Cron workflows run automatically once per day and can also be dispatched
manually.

## Configuration

Set the following repository secrets so images can be pushed:

- `QUAY_USERNAME`
- `QUAY_TOKEN`
