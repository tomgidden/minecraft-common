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
      modrinth_id: ${{ vars.MODRINTH_ID || secrets.MODRINTH_ID }}
      curseforge_id: ${{ vars.CURSEFORGE_ID || secrets.CURSEFORGE_ID }}
    secrets: inherit
```

`|| false` is needed because `inputs.*` is null on a tag push, and a null would
fail the boolean type check.

The project ids are passed explicitly because a called workflow cannot see the
caller's `vars` — only `secrets: inherit` crosses that boundary automatically.
The `vars.X || secrets.X` form means a repo can hold its ids either way.

## What each mod must provide

A `.github/release/` directory:

| file | purpose |
|---|---|
| `release.env` | `MOD_NAME`, `ARTIFACT`, `VERSION_TYPE`, `PRERELEASE`, `GAME_VERSIONS` |
| `RELEASE.md` | GitHub release body; `${VERSION}` is substituted |
| `CHANGELOG.md` | changelog for Modrinth/CurseForge; `${VERSION}` is substituted |

`ARTIFACT` must match `archives_base_name` in `gradle.properties`, since jars
are located as `<loader>/build/libs/<ARTIFACT>-<loader>-<VERSION>.jar`.

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

Every tag builds. The tag's shape decides what gets published:

| trigger | builds | GitHub release | Modrinth / CurseForge |
|---|---|---|---|
| tag `26.0.3` | yes | yes | yes |
| tag `26.0.3-pre1` | yes | yes, as a **prerelease** | no |
| tag `workflow-update-26.0.2` | yes | no | no |
| manual dispatch | yes | yes, as a **prerelease** | only if ticked |

Tag patterns are strict: `^[0-9]+\.[0-9]+(\.[0-9]+)?$` for a release and
`^[0-9]+\.[0-9]+(\.[0-9]+)?-[A-Za-z0-9_]+$` for a prerelease. `v26.0.3` and
`26.0.3-pre.1` (dot in the suffix) match neither, so they build and publish
nothing.

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
