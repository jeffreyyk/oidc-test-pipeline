# OIDC npm test pipeline for JFrog Platform

GitHub Actions workflows that test npm access to JFrog Artifactory from GitHub Actions **without stored credentials**. Each workflow exchanges a GitHub OIDC token for a short-lived JFrog access token, then runs npm operations against Artifactory in two ways:

- **JFrog CLI** (`jf npm ...`)
- **Native npm**, using the OIDC token as the registry auth token

## Workflows

| Workflow file | Purpose | Repository used | Access needed |
|---|---|---|---|
| `oidc-npm-pull-push-remote-test.yml` | Pull `lodash@4.17.21` through the npm remote repo | `jk-npm-dev-remote` | Read + Deploy/Cache |
| `oidc-npm-publish-test-suite.yml` | Publish a generated test package, then verify it with `npm view` | `jk-npm-dev-local` | Read + Deploy/Cache + Annotate |
| `oidc-npm-pull-local-test.yml` | Pull a package already published to the local repo | `jk-npm-dev-local` | Read |

Despite its name, the `pull-push-remote` workflow only tests **pulls**. Publishing is covered by the publish workflow.

All three workflows are manual (`workflow_dispatch`) and take one input:

| Input | Values | Effect |
|---|---|---|
| `env` | `nonprod` (default), `prod` | Selects the Artifactory instance: `psblr.jfrog.io` for `nonprod`, `artifactory.cba` for `prod` |

## Prerequisites

### GitHub

- The workflow files are in `.github/workflows/` on the **default branch**. `workflow_dispatch` runs only appear once the file is on that branch.
- Workflows request `id-token: write` and `contents: read`.
- The runner needs Node.js and npm. Workflows use `ubuntu-latest` (or a self-hosted runner label such as `default`, if your organization defines one).

### JFrog

1. **OIDC integration** and **identity mapping** that trust this GitHub repository (see [Identity mapping](#identity-mapping) below).
2. **Permission target(s)** covering the repositories below. The group named in the mapping's token scope must have these permissions:

   | Repository | Type | Permissions for the group |
   |---|---|---|
   | `jk-npm-dev-remote` | npm remote | Read, **Deploy/Cache** |
   | `jk-npm-dev-local` | npm local | Read, Deploy/Cache, Annotate (publish only) |

   Include pattern `**`, with no excludes that match the test packages.

> **Important:** a remote repo needs **Deploy/Cache**, not only Read. When Artifactory fetches a package from upstream, it writes it into the repo's cache on behalf of the caller. Without this permission, pulls fail even though the repo is visible.

Repositories in these tests are **not** in a JFrog Project. If you move them into a project, access is governed by project roles and the token needs a matching project-level identity mapping.

## Identity mapping

The workflows never hold a JFrog password or API key. Instead, `jfrog/setup-jfrog-cli` presents GitHub's short-lived OIDC token to JFrog, and JFrog exchanges it for a JFrog access token. Two objects in JFrog control this exchange: the **OIDC integration** (who is trusted) and the **identity mapping** (what the token can do).

![JFrog OIDC integration and identity mapping settings](docs/images/jfrog-identity-mapping.png)

### How the exchange works

1. The workflow requests an OIDC token from GitHub (this needs `permissions: id-token: write`).
2. `setup-jfrog-cli` sends it to JFrog, naming the provider (`oidc-provider-name`) and audience (`oidc-audience`).
3. JFrog checks that the token comes from the configured provider URL, was issued for the configured audience, and that its claims match an identity mapping.
4. JFrog issues an access token whose scope is defined by that mapping. All later `jf` and `npm` calls use it.
5. Artifactory then checks the token's group against the **permission targets** for each repository.

### OIDC integration settings

| Setting | Value in this setup | Notes |
|---|---|---|
| Provider name | `github-oidc-cba-general` | Must equal `OIDC_PROVIDER_NAME` in the workflows |
| Provider type | GitHub | |
| Provider URL | `https://token.actions.githubusercontent.com` | GitHub's OIDC issuer |
| Audience | `cba-github-actions` | Must equal `OIDC_AUDIENCE` in the workflows |
| Organization | `jeffreyyk` | The GitHub owner whose repositories are trusted |
| Enable permissive configuration | Off | Keep off so claims are matched strictly |

### Identity mapping settings

| Setting | Value in this setup | Notes |
|---|---|---|
| Name | `github-jeffreyyk` | |
| Priority | `9999` | Decides which mapping wins when more than one matches |
| Claims JSON | `iss`, `repository_owner`, `repository` (see below) | Token claims must match what is listed here |
| Token scope | Group | The token gets the permissions of the named group |
| Groups | `jk-cba-oidc` | This is the group that needs the repository permissions above |
| Service | All | Token can be used with all JFrog services |
| Token expiration | 1440 minutes (24 hours) | |
| Self revocable OIDC token | Off | |

The claims used here restrict token issuance to this repository:

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "repository_owner": "jeffreyyk",
  "repository": "jeffreyyk/oidc-test-pipeline"
}
```

A workflow in any other repository, even in the same organization, will not match this mapping and cannot get a token.

### Things to know

- **The group is what grants access.** The mapping only says *which group* the token represents. Repository access comes from permission targets on that group. If a mapping is correct but a job still fails with 403 or 404, the permission target is the usual cause (see [Troubleshooting](#troubleshooting)).
- **Group names must match exactly.** Names are case-sensitive. A group named in the mapping that does not exist produces a token with no permissions from it.
- **Project mappings take precedence.** The JFrog UI notes that project-level mappings override global ones. If the token scope is not what you expect, check the **Project** tab as well as **Global**.
- **Tighten claims if needed.** GitHub's token carries more claims you can add to the mapping, such as `ref` (branch), `environment`, or `workflow`, to limit tokens to specific branches, environments or workflows.
- **Consider a shorter expiry.** These workflows finish in minutes, so a 24-hour token is longer than needed. Lower the expiration, or enable self-revocable tokens, for production use.
- **One mapping serves all three workflows.** They share the same provider, audience and mapping, so a permission change applies to all of them.

## Running a workflow

1. Open **Actions** in the GitHub repository.
2. Select the workflow.
3. Click **Run workflow**, choose `nonprod` or `prod`, and run.

### What success looks like

- **Remote pull:** both jobs print an npm install success message and `added 1 package`. An `npm audit` warning about lodash is expected and unrelated to this setup.
- **Publish:** both jobs end with `Verified <package>@<version> in jk-npm-dev-local`. Each run publishes new versions (`1.0.0-run.<run_id>.<attempt>`), tagged `oidc-test`. The packages are named `oidc-npm-cli-test-<repo id>` and `oidc-npm-raw-test-<repo id>` and stay in Artifactory after the run.
- **Local pull:** both jobs print `OIDC npm publish test`, the content exported by the test package.

## Configuration notes

- **Pull-local package.** `oidc-npm-pull-local-test.yml` pins `PACKAGE_NAME` and `PACKAGE_VERSION` to a package already in `jk-npm-dev-local`. Run the publish workflow first, then update these two values to a published package and version (visible in the Artifactory tree view).
- **Pinned actions.** Third-party actions are pinned by commit SHA.
- **Token handling.** The token is masked with `::add-mask::` before it is exported or written to `.npmrc`. The publish workflow also removes `.npmrc` when it finishes.
- **Clean-up.** Delete old test packages from `jk-npm-dev-local` when you no longer need them.

## Troubleshooting

The status codes below come from `jf rt curl` checks against Artifactory.

| Symptom | Likely cause | Fix |
|---|---|---|
| `E404` on `npm install` from a remote repo, `404` on `api/npm/<repo>/<pkg>` | Token's group lacks Read or Deploy/Cache on the repo | Add the permissions in the permission target |
| `403` on a tarball, with a message like "not permitted to deploy ... into `<repo>-cache`" | Group lacks **Deploy/Cache** on the remote repo | Grant Deploy/Cache |
| `403` on publish | Group lacks Deploy/Cache or Annotate on the local repo | Grant them |
| `302` on a direct tarball request | Not an error: Artifactory redirected the download to cloud storage | Follow redirects (`curl -L`); npm does this automatically |
| `200` on `api/repositories/<repo>` but package requests fail | The config endpoint does not need Read on package contents | Check permissions; this call does not prove access |
| Token scope shows a different group than expected | A different identity mapping matched | Check mapping priority and claims (Global and Project tabs) |
| Jobs stay "Queued" | `runs-on` label has no matching runner | Use `ubuntu-latest` or a valid runner label |
| Package blocked | Curation or an Xray watch with "Block download" | Check the Curation audit log and Xray watches |

To see what the token actually carries, decode its payload (no secrets are printed) and check `sub` and `scp`. To find the exact denial reason, search the Artifactory request and service logs for the trace ID printed by the JFrog CLI.

## Repository layout

```
.github/workflows/
├── oidc-npm-pull-push-remote-test.yml   # pull via remote repo
├── oidc-npm-publish-test-suite.yml      # publish to local repo
└── oidc-npm-pull-local-test.yml         # pull from local repo
docs/images/
└── jfrog-identity-mapping.png           # screenshot used in this README
README.md
```
