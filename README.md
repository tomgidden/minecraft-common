# minecraft-mod-build-and-publish

Shared GitHub Actions pipeline for my Minecraft mods: builds the Fabric and/or
NeoForge subprojects, then publishes to GitHub Releases, Modrinth and
CurseForge as independent jobs.

This repo is public because GitHub only reliably allows cross-repository
workflow reuse from a public repo on a personal account. It contains no
secrets — only `secrets.*` references, which resolve in the *calling* repo's
context.

## Using it

Each mod repo keeps a thin stub at `.github/workflows/build-and-release.yml`:

```yaml
name: Build and Release

on:
  workflow_dispatch:
    inputs:
      modrinth:
        description: 'Also publish to Modrinth'
        type: boolean
        default: false
      curseforge:
        description: 'Also publish to CurseForge'
        type: boolean
        default: false
  push:
    tags:
      - '*'

jobs:
  call:
    uses: tomgidden/minecraft-mod-build-and-publish/.github/workflows/build-and-release.yml@v1
    permissions:
      contents: write
    with:
      modrinth: ${{ inputs.modrinth || false }}
      curseforge: ${{ inputs.curseforge || false }}
      modrinth_id: ${{ vars.MODRINTH_ID }}
      curseforge_id: ${{ vars.CURSEFORGE_ID }}
    secrets: inherit
```

`|| false` is needed because `inputs.*` is null on a tag push, and a null would
fail the boolean type check.

The project ids are passed explicitly because a called workflow cannot see the
caller's `vars` — only `secrets: inherit` crosses that boundary automatically.

If a repo holds its ids as **secrets** rather than vars, leave those two lines
out: the pipeline falls back to `secrets.MODRINTH_ID` / `secrets.CURSEFORGE_ID`
on its own side. The fallback can't live in the stub, because the `secrets`
context is not available in a caller's `with:` block — only `github`, `needs`,
`strategy`, `matrix`, `inputs` and `vars` are, and using `secrets` there fails
the whole workflow at validation time, before any job is scheduled.

## What each mod must provide

Values come from `gradle.properties`, which is already the source of truth for
the build:

| property | purpose |
|---|---|
| `mod_name` | display name, e.g. `Trade School` |
| `archives_base_name` | jar basename; jars are located as `<loader>/build/libs/<name>-<loader>-<VERSION>.jar` |
| `game_versions` | **optional** comma-separated list for CurseForge, e.g. `26.1.2,26.2,26.3`. Omit to infer from the jar's metadata |

The release channel is *not* configured — it is derived from the tag, so it
cannot drift from what is actually being released. (This replaces the old
`.github/release/release.env`; delete that file when adopting this version.)

`game_versions` is the one value that cannot be derived: CurseForge rejects any
version string it doesn't already know, and a mod spanning 26.1.2 to 26.3 also
has to list 26.2 by hand.

Plus a `.github/release/` directory for prose:

| file | purpose |
|---|---|
| `RELEASE.md` | GitHub release body; `${VERSION}` is substituted |
| `CHANGELOG.md` | changelog for Modrinth/CurseForge; `${VERSION}` is substituted |

Loaders are discovered from the repo layout — a loader is built only if its
subproject directory exists — so Fabric-only and Fabric+NeoForge mods use the
same stub.

## Secrets

Set per mod repo:

| name | kind | notes |
|---|---|---|
| `MODRINTH_ID` | var (or secret) | passed in by the stub |
| `CURSEFORGE_ID` | var (or secret) | passed in by the stub |
| `MODRINTH_TOKEN` | **secret** | via `secrets: inherit` |
| `CURSEFORGE_TOKEN` | **secret** | via `secrets: inherit` |

The ids are ordinary repo variables; the tokens must be secrets.

A missing *token* means that target is skipped, so a fork still builds and
makes a GitHub release. A token set with an empty *id* fails loudly rather than
publishing nothing.

The Modrinth token needs more than "Create Version" scope: mc-publish also
unfeatures older versions on publish, which needs edit permission. A
create-only token fails with `401 ... editing version through v3 route`.

## What publishes when

Every tag builds. Two separate questions decide the rest: the tag's **shape**
says what kind of release it is, and the **trigger** says whether it publishes.

| tag | channel | on tag push | on manual dispatch |
|---|---|---|---|
| `26.0.3` | release | GitHub + Modrinth + CurseForge | per tick boxes |
| `26.0.3+foo` | release | GitHub only | per tick boxes |
| `26.0.3-beta1` | beta, **prerelease** | GitHub only | per tick boxes |
| `26.0.3-alpha1` | alpha, **prerelease** | GitHub only | per tick boxes |
| `workflow-update-26.0.2` | — | build only | build + GitHub |

**A bare `26.0.3` is the only thing that ever publishes by itself.**

Tag patterns are strict: `^[0-9]+\.[0-9]+(\.[0-9]+)?$` for a release,
with `+[A-Za-z0-9_.-]+` or `-[A-Za-z0-9_.]+` for the two suffixed forms.
`v26.0.3` matches none of them, so it builds and publishes nothing.

The channel sent to the mod sites is derived from the `-` suffix: `beta` for
`-beta*`, `-rc*` and `-pre*`; `alpha` for `-alpha*` or any unrecognised suffix,
since understating maturity is the harmless direction to fail.

`+foo` is an **exceptional post-release patch** — a release-bug fix or very
minor tweak, published by hand after deleting the base release. It counts as a
real release rather than a prerelease, but never goes out automatically:
pushing the tag only builds and makes a GitHub release, and reaching the mod
sites takes a manual dispatch with the boxes ticked. (Semver says build
metadata is ignored for precedence, but neither Modrinth nor CurseForge
documents what it does with the string, so it stays under manual control.)

A dispatch from a **branch** has no tag to be judged by, so it stays a
prerelease and takes mc-publish's default channel — a test build never looks
like a real one.

On a version tag the **tag is authoritative**: the build passes
`-Pmod_version=<tag>`, so jar names always match what the release jobs look
for. A differing `mod_version` in `gradle.properties` is a warning, not an
error.

## Versioning this repo

Callers pin `@v1`. Move that tag for backward-compatible changes:

```bash
git tag -fa v1 -m "..." && git push -f origin v1
```

Breaking changes to the inputs get a `v2` and a per-repo stub update.

## Notes

- Publishing is split into independent jobs, so a failed Modrinth publish can
  be retried with "Re-run failed jobs" without rebuilding. Re-running an *old*
  run replays that run's workflow files, so a fix here needs a new run.
- mc-publish infers loaders and game versions from a single jar's metadata per
  call, so the publish jobs fan out over a matrix of built loaders, one jar per
  call. Passing several loaders' jars to one call silently publishes only one.
- CurseForge holds new uploads in an "Under Review" queue; a "Successfully
  published" log is accurate even when the public API doesn't show the file yet.
