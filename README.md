# OIDC test pipeline for JFrog Platform

## Additional package status

npm was the first package type. These seven are the only packages added on top of it. Each has a **pull** workflow (through the remote repo) and a **publish** workflow (to the local repo). **Both** means the workflow has a JFrog CLI job and a native client job. **Native only** means there is no JFrog CLI job.

| Package | Status | Clients covered |
|---|---|---|
| Conan | Done | Native only |
| PyPI (Python) | Done | Both |
| Docker | Done | Native only |
| Go | Done | Both |
| Gradle | Done | Both |
| NuGet | Done | Both |
| Maven | Done | Both |

Other Artifactory package types are not covered yet.

GitHub Actions workflows that test access to JFrog Artifactory from GitHub Actions **without stored credentials**. Each workflow exchanges a GitHub OIDC token for a short-lived JFrog access token, then runs package operations against Artifactory in up to two ways:

- **JFrog CLI**, using the wrapper for the package manager (`jf npm`, `jf mvn`, `jf go` and similar)
- **Native client** (npm, Maven, pip and so on), using the OIDC token as the credential

## Workflows

### npm

| Workflow file | Purpose | Repository used | Access needed |
|---|---|---|---|
| `oidc-npm-pull-remote-test.yml` | Pull `lodash@4.17.21` through the npm remote repo | `jk-npm-dev-remote` | Read + Deploy/Cache |
| `oidc-npm-publish-local-test.yml` | Publish a generated test package, then verify it with `npm view` | `jk-npm-dev-local` | Read + Deploy/Cache + Annotate |
| `oidc-npm-pull-local-test.yml` | Pull a package already published to the local repo | `jk-npm-dev-local` | Read |
| `oidc-npm-pull-push-remote-debug.yml` | Debug version of the remote pull test, used while troubleshooting access | `jk-npm-dev-remote` | Read + Deploy/Cache |

### Additional packages

| Package | Pull workflow (remote repo) | Publish workflow (local repo) |
|---|---|---|
| Conan | `oidc-conan-pull-remote-test.yml` | `oidc-conan-publish-local-test.yml` |
| PyPI (Python) | `oidc-pypi-pull-remote-test.yml` | `oidc-pypi-publish-local-test.yml` |
| Docker | `oidc-docker-pull-remote-test.yml` | `oidc-docker-publish-local-test.yml` |
| Go | `oidc-go-pull-remote-test.yml` | `oidc-go-publish-local-test.yml` |
| Gradle | `oidc-gradle-pull-remote-test.yml` | `oidc-gradle-publish-local-test.yml` |
| NuGet | `oidc-nuget-pull-remote-test.yml` | `oidc-nuget-publish-local-test.yml` |
| Maven | `oidc-maven-pull-remote-test.yml` | `oidc-maven-publish-local-test.yml` |

See [Additional package workflows](#additional-package-workflows) for the repositories and tests each one uses.

All workflows are manual (`workflow_dispatch`) and take one input:

| Input | Values | Effect |
|---|---|---|
| `env` | `nonprod` (default), `prod` | Selects the Artifactory instance: `psblr.jfrog.io` for `nonprod`, `artifactory.cba` for `prod` |

## Prerequisites

### GitHub

- The workflow files are in `.github/workflows/` on the **default branch**. `workflow_dispatch` runs only appear once the file is on that branch.
- Workflows request `id-token: write` and `contents: read`.
- Workflows use `ubuntu-latest` (or a self-hosted runner label such as `default`, if your organization defines one). The tools each workflow needs (Node.js, Java, Maven, Gradle, .NET, Python, Go, Docker) are preinstalled there. Conan is installed by the workflow.

### JFrog

1. **OIDC integration** and **identity mapping** that trust this GitHub repository (see [Identity mapping](#identity-mapping) below).
2. **Permission target(s)** covering the repositories below. The group named in the mapping's token scope must have these permissions:

   | Repository | Type | Permissions for the group |
   |---|---|---|
   | `jk-npm-dev-remote` | npm remote | Read, **Deploy/Cache** |
   | `jk-npm-dev-local` | npm local | Read, Deploy/Cache, Annotate (publish only) |
   | `jk-<package>-dev-remote` | remote, one per additional package | Read, **Deploy/Cache** |
   | `jk-<package>-dev-local` | local, one per additional package | Read, Deploy/Cache, Annotate (publish only) |

   Go pulls also need a Go **virtual** repo that includes the remote. Give the group Read on it.

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
<img width="1670" height="906" alt="image" src="https://github.com/user-attachments/assets/ced71aaa-9066-42e2-8059-c73576d7bd87" />


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
- **One mapping serves all workflows.** They share the same provider, audience and mapping, so a permission change applies to all of them.

## Additional package workflows

Each additional package has two workflows, named `oidc-<package>-pull-remote-test.yml` and `oidc-<package>-publish-local-test.yml`.

**Pull workflows** log in with OIDC, check that the remote repo exists and has the expected package type, download a known file directly through the repo (expecting HTTP 200), and then pull a small package with the client. **Publish workflows** build a small test package, publish it to the local repo, and verify it is stored. Versions include the run ID and attempt, so reruns never collide. Test packages stay in Artifactory.

| Package | Remote repo | Local repo | Pull test | Publish test |
|---|---|---|---|---|
| Conan | `jk-conan-dev-remote` | `jk-conan-dev-local` | Download the `zlib/1.3.1` recipe | `conan create` and `conan upload` |
| PyPI (Python) | `jk-pypi-dev-remote` | `jk-pypi-dev-local` | Download `requests==2.31.0` | Build the package and upload with Twine |
| Docker | `jk-docker-dev-remote` | `jk-docker-dev-local` | Pull `hello-world` | Build, push, and pull back an image |
| Go | `jk-go-dev-remote` | `jk-go-dev-local` | Download `github.com/pkg/errors@v0.9.1` | Publish a generated module |
| Gradle | `jk-gradle-dev-remote` | `jk-gradle-dev-local` | Resolve `junit:junit:4.13.2` | Publish a generated jar |
| NuGet | `jk-nuget-dev-remote` | `jk-nuget-dev-local` | Restore `Newtonsoft.Json` 13.0.3 | Pack and push a generated package |
| Maven | `jk-maven-dev-remote` | `jk-maven-dev-local` | Resolve `junit:junit:4.13.2` | Deploy a generated jar |

Repository keys are set at the top of each file (`REMOTE_REPO` and `LOCAL_REPO`). Edit them if yours differ.

**Package-specific notes**

- **Go:** pulls use `RESOLVE_REPO`. Artifactory rejects the Go API on a remote repo (HTTP 400, "cannot be used without a virtual repository"), so `RESOLVE_REPO` should be a Go virtual repo that includes `jk-go-dev-remote`.
- **Conan:** the remote repo's upstream URL must be `https://center2.conan.io`, the Conan 2 ConanCenter.
- **Maven and Gradle:** the pull tests assume the remote repo mirrors Maven Central.
- **JFrog CLI jobs:** commands such as `jf mvn`, `jf go` and `jf pip` collect build-info when a build name is set. The workflows clear the build name and number for those commands so no build-info collection is attempted.
- **Basic-auth clients** (Docker, pip, NuGet and similar) use the `oidc-user` and `oidc-token` outputs of `setup-jfrog-cli`.
- **Permissions:** the group needs Read and Deploy/Cache on each remote repo, and Read, Deploy/Cache and Annotate on each local repo.

## Running a workflow

1. Open **Actions** in the GitHub repository.
2. Select the workflow.
3. Click **Run workflow**, choose `nonprod` or `prod`, and run.

### What success looks like

- **Remote pull:** both jobs print an npm install success message and `added 1 package`. An `npm audit` warning about lodash is expected and unrelated to this setup.
- **Publish:** both jobs end with `Verified <package>@<version> in jk-npm-dev-local`. Each run publishes new versions (`1.0.0-run.<run_id>.<attempt>`), tagged `oidc-test`. The packages are named `oidc-npm-cli-test-<repo id>` and `oidc-npm-raw-test-<repo id>` and stay in Artifactory after the run.
- **Local pull:** both jobs print `OIDC npm publish test`, the content exported by the test package.
- **Additional packages:** each job ends green and writes a job summary (target environment, repository and the paths or packages it verified) to the run page.

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
├── oidc-conan-publish-local-test.yml
├── oidc-conan-pull-remote-test.yml
├── oidc-docker-publish-local-test.yml
├── oidc-docker-pull-remote-test.yml
├── oidc-go-publish-local-test.yml
├── oidc-go-pull-remote-test.yml
├── oidc-gradle-publish-local-test.yml
├── oidc-gradle-pull-remote-test.yml
├── oidc-maven-publish-local-test.yml
├── oidc-maven-pull-remote-test.yml
├── oidc-npm-publish-local-test.yml
├── oidc-npm-pull-local-test.yml
├── oidc-npm-pull-push-remote-debug.yml
├── oidc-npm-pull-remote-test.yml
├── oidc-nuget-publish-local-test.yml
├── oidc-nuget-pull-remote-test.yml
├── oidc-pypi-publish-local-test.yml
└── oidc-pypi-pull-remote-test.yml
docs/images/
└── jfrog-identity-mapping.png           # screenshot used in this README
README.md
```
