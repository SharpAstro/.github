# CLAUDE.md — the SharpAstro org root

This file lives in `github.com/SharpAstro/.github`, cloned as `.github/` inside the
org root, and is imported by the org root's one-line `CLAUDE.md`. It is here so it is
version-controlled and reviewable rather than sitting untracked on one machine —
edit it here. Every path below is relative to the **org root**, one level up.

## The org root is not a repository

It is an **org root**: 23 sibling clones of `github.com/SharpAstro` — 22 code repos
plus this `.github` repo, which carries the org profile README (`profile/README.md`),
this file, and the pattern doc below. `git` commands run at the root itself fail —
`cd` into a repo first. Commit identity comes from `~/.gitconfig`, which maps the org
root path to `sebastian.godelet@outlook.com` via an `includeIf gitdir:` rule.

Most repos carry their own `CLAUDE.md` covering architecture, namespaces and
repo-specific commands — **read it first and defer to it.** (Absent in `FITS.Lib`,
`FreeTypeBindings`, `Lzip.Lib`, `QHYCCD.SDK`, `SER.Lib`, `cache-apt-pkgs-action`,
`openphd2-multiscope`, `skyfi-forwarder`, `sofa-c-mirror`.)

## Build & test

No `global.json` anywhere; repos target `net10.0` (several also `netstandard2.0`) and
CI pins `setup-dotnet` to `10.0.x`. The solution sits at the repo root or under `src/`,
as `.sln` or `.slnx` — both are in use.

```bash
dotnet build                                                  # from the repo root
dotnet test src/<Name>.Tests
dotnet test src/<Name>.Tests --filter "FullyQualifiedName~TestMethodName"
```

Tests are **xunit.v3** throughout; `FITS.Lib` is the sole NUnit holdout. `QHYCCD.SDK`,
`TianWen.DAL`, `canon-edsdk` and `zwo-sdk-nuget` publish packages but have no test step.

## Local builds do not use the pinned packages

The one cross-repo fact that changes how any build behaves here. Eight repos
(`Console.Lib`, `DIR.Lib`, `FC.SDK`, `Fonts.Lib`, `LALR.CC`, `SdlVulkan.Renderer`,
`WebGl.Renderer`, `tianwen`) carry `UseLocal*` properties that **self-enable** via
`Exists()` when the sibling clone is present — so a local build compiles against
sibling *source* and the `PackageReference` version is never exercised.

- Editing a sibling changes downstream builds with no version bump anywhere. When a
  change spans repos, build the consumer too before assuming it is done.
- To reproduce what CI actually restores: `dotnet build -p:UseLocalDirLib=false`
  (substitute the relevant switch).

## Versioning

`<VersionMajorMinor>` in `Directory.Build.props` (sometimes under `src/`) is the only
place `MAJOR.MINOR` is written; CI reads it back and appends the run number. A release
bump is that one line — `1.0 → 1.1` additive, `1.x → 2.x` breaking. Everything else
(`VersionPrefix`, `AssemblyVersion`, the workflow) derives from it and must not be
hand-edited. `Lzip.Lib` is the reference implementation.

## Fuller pattern write-up

`.github/docs/dotnet-ci-pattern.md` — in this repo, beside this file — holds the rest:
the CI workflow skeleton, the `NUGET_API_KEY` org-secret requirement, the per-repo
conformance checklist, the `UseLocal*` switch list and its two `..\..\..` depth
conventions, which projects actually propagate a dependency, the known
package-id collisions (`Canon.EDSDK`, `FreeTypeSharp.UWP`), orphaned-CPM-entry
examples, and the default-branch drift (`master` for `Codecs`, `FITS.Lib`, `LALR.CC`;
`LALR.CC` publishes on a `v*` tag, not on push).

Read it before: adding or re-scaffolding a repo's CI, bumping a dependency pin,
publishing, or judging whether a version looks wrong.

The `repos` skill (`/repos`) reports across all sibling repos at once — git/dirty/sync
state and build-pattern conformance, pin currency vs published and about-to-be-published
versions, pending work in dependency order, and open PRs with check status. It is a
user-level skill (`~/.claude/skills/repos/`), not part of this repo, because it serves
several org roots; this repo only supplies the SharpAstro pattern doc it reads. Prefer
it over hand-rolling a sweep; use its `worklist.sh` for release order rather than
inferring one from package names.
