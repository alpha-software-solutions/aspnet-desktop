# Releasing

Releases are automated with [release-please](https://github.com/googleapis/release-please) and driven by [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/). Nobody picks version numbers or writes tags by hand.

## Components

The repository releases two components, each with its own version, tag, changelog and packages:

| Component | Packages | Changes counted | Tag | Changelog |
|---|---|---|---|---|
| `AspNetDesktop` | `AlphaSoftware.AspNetDesktop`, `.Hosting`, `.Windows`, `.Mac`, `.Linux`, `.Templates` (NuGet) | everything outside `src/client` | `AspNetDesktop-v1.2.3` | [`CHANGELOG.md`](../CHANGELOG.md) |
| `aspnetdesktop-client` | `@alpha-software/aspnetdesktop-client` (npm) | `src/client` | `aspnetdesktop-client-v1.2.3` | [`src/client/CHANGELOG.md`](../src/client/CHANGELOG.md) |

A change to the npm client releases only the npm package, and a change to the .NET side (including the samples the templates are built from) releases only the NuGet packages. A commit that touches both counts for both.

The templates package pins the client to the version in `src/client/package.json` at the time of the NuGet release (`^x.y.z`). When both components have a release PR open, merge the client's first and let it publish, so the templates never reference a client version that isn't on npmjs.com yet.

## How a release happens

1. Commits land on `master`. Only `feat`, `fix`, `perf`, `revert` and breaking changes (`!` or a `BREAKING CHANGE:` footer) are releasable; `docs`, `chore`, `ci`, `build`, `test`, `refactor` and `style` never trigger a release on their own.
2. On every push to `master`, the **Release** workflow runs release-please. For each component with releasable changes, it opens or updates a release PR (`chore(master): release AspNetDesktop 0.2.0`) that bumps the version and adds the changelog entry.
3. Merge the release PR when you want to ship. You can edit its changelog first.
4. release-please tags the merge commit and creates the GitHub Release. The same workflow run then runs CI and publishes that component's packages, and attaches them to the release.

Versions follow [SemVer](https://semver.org/spec/v2.0.0.html). Before 1.0, `feat` and breaking changes bump the minor version and `fix` the patch; from 1.0, breaking changes bump the major. To force a version, add a `Release-As: 1.0.0` footer to a commit.

Dependabot's commits use the `build(deps)` and `ci(deps)` prefixes, so dependency updates ship with the next release rather than triggering one. To release a dependency update on its own (e.g. a security fix), merge it and push a `fix(deps): ...` commit describing it.

Between releases, pushes to `master` that affect the packages build them as previews of the next patch (`0.1.1-preview.0.5` is five commits after `AspNetDesktop-v0.1.0`, via [MinVer](https://github.com/adamralph/minver)) and upload them as workflow artifacts. They are not published.

## Release PRs and CI

CI doesn't build or test release PRs. They only change versions and changelogs, and the Release workflow runs the full CI before it publishes anything. Instead, each Release run reports the required **CI result** check on the open release PRs, so you can merge one as soon as you want to ship.

- The `AspNetDesktop` release PR only touches release-please's files, which CI ignores, so no workflow runs for it at all.
- The `aspnetdesktop-client` release PR also bumps the version in `src/client/package.json` and `package-lock.json`, so GitHub creates a CI run for it. Release PRs are opened by `github-actions[bot]` with the workflow's own `GITHUB_TOKEN`, and GitHub holds runs for such PRs until a maintainer approves them. You don't need to approve it; if you do, it skips every job.

If anything besides versions and changelogs is pushed to a release PR, CI checks it like any other pull request, and the Release workflow no longer reports **CI result** for it.

To have the client release PR's run start without waiting for approval, give release-please a token whose PRs trigger workflows normally. Create a [fine-grained personal access token](https://github.com/settings/personal-access-tokens/new) for this repository with **Contents**, **Pull requests** and **Issues** set to read and write, then:

```bash
gh secret set RELEASE_PLEASE_TOKEN --repo alpha-software-solutions/aspnet-desktop
```

## Where packages go

| Registry | Packages | Setup |
|---|---|---|
| npmjs.com | `@alpha-software/aspnetdesktop-client` | always; [credentials below](#npm) |
| nuget.org | `AlphaSoftware.AspNetDesktop*` | opt-in with `PUBLISH_NUGET_ORG`; [credentials below](#nuget-pick-one) |
| GitHub Packages | `AlphaSoftware.AspNetDesktop*` | always; nothing to set up |

Every release also gets its packages (`.nupkg`, `.snupkg`, `.tgz`) attached to the GitHub Release.

The publish jobs for npmjs.com and nuget.org run in the `release` environment, which only accepts deployments from `master`. To put releases that already exist on a newly enabled registry, publish them again, the client first. Versions that are already on a registry are skipped.

```bash
gh workflow run release.yml --repo alpha-software-solutions/aspnet-desktop -f publish=aspnetdesktop-client
gh workflow run release.yml --repo alpha-software-solutions/aspnet-desktop -f publish=AspNetDesktop
```

### GitHub Packages (NuGet only)

The workflow publishes the NuGet packages with its own `GITHUB_TOKEN`. They're linked to this repository through `RepositoryUrl` and are public because the repository is.

- **Consumers need a token** with `read:packages`, even for public packages. Once per machine, create a [personal access token (classic)](https://github.com/settings/tokens/new?scopes=read:packages&description=AspNetDesktop%20packages) with that scope and register it:
  ```bash
  # On Windows you can leave out --store-password-in-clear-text (the password is encrypted there).
  dotnet nuget add source https://nuget.pkg.github.com/alpha-software-solutions/index.json --name aspnetdesktop --username <github-user> --password <token> --store-password-in-clear-text
  ```
- **No symbol server**: GitHub Packages doesn't host `.snupkg` files. They're attached to the GitHub Release, and Source Link still maps to the sources.
- **No npm client**: GitHub Packages only hosts npm packages under the repository owner's scope (`@alpha-software-solutions`), which would give the client a second name. It's published to npmjs.com only. Versions up to 0.1.1 are still on GitHub Packages as `@armanossiloko/aspnetdesktop-client`.

### nuget.org

Turn it on once its credentials (below) are in place:

```bash
gh variable set PUBLISH_NUGET_ORG --repo alpha-software-solutions/aspnet-desktop --body true
```

#### NuGet (pick one)

**Trusted publishing (recommended, no long-lived secrets)**

1. On nuget.org, go to **Trusted Publishing** → **Create**, with the **AlphaSoftware** organization as the policy owner (so new package IDs are created under it) and:
   - Repository owner: `alpha-software-solutions`
   - Repository: `aspnet-desktop`
   - Workflow file: `release.yml`
   - Environment: `release`
2. Store your nuget.org **username** (not email) as a repository variable:
   ```bash
   gh variable set NUGET_USER --repo alpha-software-solutions/aspnet-desktop --body "<nuget.org username>"
   ```

**API key**

Create a key on nuget.org with *Push new packages and package versions* for the glob `AlphaSoftware.AspNetDesktop*`, then:

```bash
gh secret set NUGET_API_KEY --repo alpha-software-solutions/aspnet-desktop --env release
```

When `NUGET_USER` is set, the workflow uses trusted publishing; otherwise it falls back to `NUGET_API_KEY`.

### npmjs.com

The client is published as `@alpha-software/aspnetdesktop-client`, in the [`alpha-software`](https://www.npmjs.com/org/alpha-software) npm organization. Up to 0.1.1 it was the unscoped `aspnetdesktop-client`, which is deprecated.

#### npm

npm trusted publishing can only be configured once the package exists, so the first release of a new package needs a token:

1. On npmjs.com, go to your avatar → **Access Tokens** → **Generate New Token**, and create a token with *Bypass two-factor authentication* enabled, **Packages and scopes** set to *Read and write* for the `@alpha-software` scope (the package doesn't exist yet, so you can't select it), and a short expiration. Then:
   ```bash
   gh secret set NPM_TOKEN --repo alpha-software-solutions/aspnet-desktop --env release
   ```
2. A new trusted publisher expires unless a publish uses it within 2 days, and publishing an existing version again doesn't count (it's skipped). So add it just before you merge the next client release PR: open the package on npmjs.com → **Settings** → **Trusted Publisher** → GitHub Actions with owner `alpha-software-solutions`, repository `aspnet-desktop`, workflow `release.yml`, environment `release`. npm prefers trusted publishing over the token when both are set. Once that release is out, delete the `NPM_TOKEN` secret and the token, and optionally set the package to require 2FA and disallow tokens. If the configuration expired, delete it and add it again.

## When something fails

The GitHub Release and tag exist as soon as the release PR is merged; packages follow once CI passes. If a later job fails:

- **Something outside the workflow** (a registry outage, missing credentials): fix it and re-run the failed jobs from the workflow run.
- **The workflow itself**: re-runs use the workflow file of the original commit, so fix it on `master`, then run **Release** manually with **publish** set to the component (`gh workflow run release.yml -f publish=aspnetdesktop-client`). That builds the component's latest release again from its tag and publishes it.

Pushes use `--skip-duplicate`, so packages that already went out are skipped. Don't delete the tag: release-please uses it to find the last release.

## Configuration files

- `release-please-config.json`: the components and versioning rules.
- `.release-please-manifest.json`: the last released version of each component. release-please updates it in every release PR; don't edit it by hand.
