# minecraft-common

A repo for shared code and resources used by my Minecraft mods.

## GitHub Actions

Shared GitHub Actions pipeline: builds the *Fabric* and/or
*NeoForge* subprojects, then publishes to *GitHub Releases*, *Modrinth* and
*CurseForge* as independent jobs.

### Using it

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
    uses: tomgidden/minecraft-common/.github/workflows/build-and-release.yml@v1
    permissions:
      contents: write
    with:
      modrinth: ${{ inputs.modrinth || false }}
      curseforge: ${{ inputs.curseforge || false }}
      modrinth_id: ${{ vars.MODRINTH_ID }}
      curseforge_id: ${{ vars.CURSEFORGE_ID }}
    secrets: inherit
```

## Gradle conventions

Shared build logic lives in `gradle/` as a precompiled convention plugin, so
each mod's root `build.gradle` is five lines:

```groovy
plugins {
    id 'cx.gid.minecraft.mod'
}
```

It supplies the Java toolchain, jar naming, the publish-version check, and the
`deploy` and `distclean` tasks. A mod needing something extra (an additional
dependency, say) adds it alongside, rather than editing a copy.

Mods pick it up as an **included build**, in `settings.gradle`:

```groovy
pluginManagement {
    def conventions = file('../common/gradle')
    if (conventions.isDirectory()) {
        includeBuild conventions
    }
    repositories { /* ... */ }
}
```

It is compiled from source, so an edit here reaches every mod immediately with
nothing to publish or version — but it does require this repo checked out
beside the mod, as `../common`. The guard keeps a standalone clone from failing
in `settings.gradle`; it then fails on the missing plugin id instead, which
says what is wrong.

## What each mod must provide

Values come from `gradle.properties`, which is already the source of truth for
the build:

| property | required | purpose |
|---|---|---|
| `mod_id` | yes | registry id, baked into saved worlds; also names the jars |
| `mod_name` | yes | display name, e.g. `Trade School` |
| `mod_group` | yes | Maven group, e.g. `cx.gid.minecraft` |
| `mod_version` | yes | default version; the tag overrides it on a version tag |
| `minecraft_version` | yes | what the mod is compiled against |
| `minecraft_version_min` | yes | oldest version the mod will run on |
| `minecraft_publish_for_versions` | yes | which versions to publish for |
| `archives_base_name` | no | override for the jar basename, if it must differ from `mod_id` |

### The three Minecraft versions

They look redundant and are not — each answers a different question, for a
different consumer:

| property | consumer | means | when wrong |
|---|---|---|---|
| `minecraft_version` | Gradle | **compiled against** — the MC jar Loom/NeoForm fetches | build fails, or you compile against the wrong API |
| `minecraft_version_min` | the mod manifests | **will run on** — templated into `fabric.mod.json` as `>=26.3 <27` | the loader refuses the mod, or runs it into a crash |
| `minecraft_publish_for_versions` | Modrinth + CurseForge | **advertised for** — the storefront claim | users download a jar that won't run |

Only the third is a claim; the first two are enforced by tooling and by the
game. It is one property for both sites: mc-publish resolves whatever it is
given into each site's own tags, so there is nothing platform-specific to
split. Entries are comma-separated, and may be exact versions or bounded
ranges (`[26.1.2,26.3]`).

It cannot be derived from the other two, which is why it is written by hand:

- the manifest range is **open-ended** (`<27`) on purpose, so a point release
  doesn't lock the mod out. As a storefront claim that would assert support for
  versions that don't exist yet — the same unchanged jar would start
  advertising 26.4 the day 26.4 ships;
- the **middle** of a span can't be guessed: expanding `>=26.1.2 <27` into
  `26.1.2, 26.2, 26.3` needs a table of every MC release, which Gradle hasn't
  got.

Being hand-written, it can fall behind — so the build fails if it doesn't cover
both `minecraft_version` and `minecraft_version_min`. The middle is left alone,
since Gradle can't know what belongs there.

```
minecraft_version_min=26.1.2   minecraft_version=26.3

26.1.2,26.2,26.3    ok
[26.1.2,26.3]       ok      range covers both ends
26.3                fails   excludes 26.1.2
26.1.2,26.2         fails   excludes 26.3
```

### Jar names

Jars are located as:

```
<loader>/build/libs/<archives_base_name ?: mod_id>-<loader>-<VERSION>.jar
```

The same fallback applies in the convention plugin, which names the jars, and
in the workflow, which globs for them — the two must agree. `mod_id` is the
default because it is the one identifier a mod cannot change, and the
hyphen-free form is the better jar name anyway.

It is deliberately **not** Gradle's own default, `rootProject.name`: that is
the directory name, often hyphenated where `mod_id` is not (`beacon-obscura`
vs `beaconobscura`). Taking it would have the release jobs glob for a jar the
build never produced — and an unmatched glob yields an empty file list rather
than an error, so the run would "succeed" having published nothing.

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

**A bare `26.0.3` is the only thing that ever publishes to GitHub, Modrinth and
CurseForge by default.**  Other tag formats need to be manually dispatched to publish.

IDs starting with `v`, eg. `v26.0.0` are not release candidates to can be used
to mark development branches on which release tags may or may not be added.

The channel sent to the mod sites is derived from the `-` suffix: `beta` for
`-beta*`, `-rc*` and `-pre*`; `alpha` for `-alpha*` or any unrecognised suffix
and `release` for a plain numeric version, like `26.0.3`.

`+foo` is an **exceptional post-release patch** — a hotfix or very
minor tweak, published by hand after deleting the base release. It counts as a
real release rather than a prerelease, but never goes out automatically:
pushing the tag only builds and makes a GitHub release, and reaching the mod
sites takes a manual dispatch with the boxes ticked.

A dispatch from a **branch** has no tag to be judged by, so it stays a
prerelease and takes mc-publish's default channel — a test build never looks
like a real one.

On a version tag the **tag is authoritative**: the build passes
`-Pmod_version=<tag>`, so jar names always match what the release jobs look
for. A differing `mod_version` in `gradle.properties` is a warning, not an
error.

## Build and release are separate

`build` uploads the jars; the three release jobs each download them. A failed
Modrinth publish retries with "Re-run failed jobs" without rebuilding.

Three useful consequences:

**Build without releasing anything.** Any tag that isn't a version — `ci-test-1`,
`workflow-update-26.0.2` — builds both loaders and publishes nothing, not even a
GitHub release. Useful for exercising CI.

**A prerelease tag releases to GitHub only.** `26.0.3-beta1` builds and makes a
GitHub prerelease; the mod sites are left alone until you ask.

**Publishing an existing release, without rebuilding.** Dispatch from the tag
with the mod-site boxes ticked *and* **Reuse the jars already on this tag's
GitHub release** ticked. `build` is skipped entirely, and the publish jobs
download the jars from that GitHub release instead.

That matters beyond saving a few minutes: a rebuild between prerelease and
publish means the bytes on Modrinth are not provably the bytes you playtested —
a toolchain snapshot or a dependency resolving differently would change them
silently. Reusing the release's own jars makes what you publish exactly what
users could already download.

Left unticked, the dispatch rebuilds first, which is the right choice when the
tag has moved.

## Versioning this repo

Callers pin `@v1`. Move that tag for backward-compatible changes:

```bash
git tag -fa v1 -m "..." && git push -f origin v1
```

Breaking changes to the inputs get a `v2` and a per-repo stub update.

## Notes

* Publishing is split into independent jobs, so a failed Modrinth publish can
  be retried with "Re-run failed jobs" without rebuilding. Re-running an *old*
  run replays that run's workflow files, so a fix here needs a new run.

* `mc-publish` infers loaders and game versions from a single jar's metadata per
  call, so the publish jobs fan out over a matrix of built loaders, one jar per
  call. Passing several loaders' jars to one call silently publishes only one.

* CurseForge holds new uploads in an "Under Review" queue; a "Successfully
  published" log is accurate even when the public API doesn't show the file yet.
