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
3. In `workspaces/<workspace>` runs `yarn install --immutable`, `yarn tsc:full`,
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
| `workspace` | yes      | Workspace folder under `workspaces/`         |
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

| Prefix              | Repository                                              | Cron schedule | Image name prefix |
| ------------------- | ------------------------------------------------------ | ------------- | ----------------- |
| `bcp-*`             | `backstage/community-plugins`                          | `0 3 * * *`   | _(none)_          |
| `rhdh-plugins-*`    | `redhat-developer/rhdh-plugins`                        | `0 4 * * *`   | _(none)_          |
| `proberaum-*`       | `proberaum/backstage-plugins`                          | `0 5 * * *`   | `proberaum-`      |

Cron matrices:

- **community-plugins:** acr, argocd, bookmarks, jfrog-artifactory,
  multi-source-security-viewer, nexus-repository-manager, npm, ocm, quay, rbac,
  servicenow, tekton, topology
- **rhdh-plugins:** adoption-insights, ai-integrations, app-defaults,
  bulk-import, extensions, global-header, homepage, lightspeed, quickstart,
  scorecard, theme, translations
- **proberaum:** analytics-viewer, config-viewer, devtools, env-viewer,
  github-notifications, hcloud, icon-viewer, permissions-viewer,
  scheduler-notifications, whats-new

## Usage

To build a plugin on demand, run the matching `*-main` or `*-pr` workflow from
the GitHub Actions tab and provide the `workspace` (and `pr` for PR builds).
Cron workflows run automatically once per day and can also be dispatched
manually.

## Configuration

Set the following repository secrets so images can be pushed:

- `QUAY_USERNAME`
- `QUAY_TOKEN`
