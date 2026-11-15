# Pinning the org's packages

How a repo pins the SharpAstro libraries it builds on (DIR.Lib, Console.Lib, SdlVulkan.Renderer, WebGl.Renderer,
AppShell, and the rest), how to tell whether the pins are right, and what to do when they are not. The mechanics of a
release (which repo bumps, in which order, how CI publishes) are in [dotnet-ci-pattern.md](dotnet-ci-pattern.md).

## The rule

1. **Pins float within a minor**: `X.Y.*`, in `Directory.Packages.props`. A CI rebuild of the same `X.Y` arrives on the
   next restore; moving to a new `X.Y` is a one-line commit that says why.
2. **Every library an app loads is built against the shared dependencies the app pins**, or an *additive* step behind
   them. The app's pin wins at restore: NuGet loads one DIR.Lib for the whole process, so a library built against an
   older DIR.Lib runs on the app's. Across additive minors that is fine. **Across a major it is a bug waiting**: a
   member the new major removed fails at run time (`MissingMethodException`), never at build time, and only on the
   code path that calls it.
3. **The train runs to the end.** When DIR.Lib moves, every library above it repins and releases, and only then the
   apps. A library left behind is invisible: the app still builds and its tests still pass.

## Checking it

In CI, as a job of its own (no secrets, no sibling clones; this repo is public, so a repo outside the org can call it):

```yaml
  pins:
    uses: SharpAstro/.github/.github/workflows/check-pins.yml@main
    # with:
    #   props: src/Directory.Packages.props   # default: Directory.Packages.props
    #   fail-on-major: true                   # default: false (warn and report only)
```

It writes two tables into the run's summary and warns on a MAJOR gap; `fail-on-major` makes that a failure. Locally,
from the org root:

```bash
python .github/scripts/check-pins.py <repo>/Directory.Packages.props            # what CI reports
python .github/scripts/check-pins.py <repo>/Directory.Packages.props --majors   # also newer MAJORs (deliberate upgrades)
```

It asks nuget.org two things for every pin:

- **Behind**: is there a newer `Major.Minor` within the pin's major? A routine bump. A newer major (the .NET 11 packages
  for a .NET 10 app, say) is never routine, so it is only listed with `--majors`.
- **Built against vs loaded with**: each pinned package's *published* `.nuspec` lists the versions it was built
  against. Where the repo pins one of those dependencies too, the two are compared. A **MAJOR** gap exits 1; a
  **minor** gap is reported, and is fine when the dependency's CHANGELOG calls the steps between additive.

## What to do with each finding

| Finding | Do |
|---|---|
| An org pin is behind | Read the library's CHANGELOG from your version up; repin; build and test; commit with what changed. |
| A MAJOR gap (a library built against an older major of a dependency the repo pins) | Repin **that library** to the repo's version, build and test it against the package, release it, then move the repo's pin to the release. |
| A minor gap | Fine if the dependency's CHANGELOG calls every step additive; repin the library at its next release anyway. |
| A third-party pin is behind | Not part of the train. If an org library pins it too (Vortice in SdlVulkan.Renderer), bump it there first. |

## Testing a library's repin the way CI will see it

- **Build against the package, not the sibling source.** A library whose sibling clone sits next to it references the
  source (`UseLocalDirLib`, `UseLocalSiblings`; see "Local sibling checkouts override packages" in the pattern doc).
  Pass `-p:UseLocalDirLib=false` so the restore takes the *published* package, the one consumers will load, and check
  `obj/project.assets.json` names the version you meant.
- **nuget.org lists a package before its search index serves it.** For several minutes after a push,
  `v3-flatcontainer/<id>/index.json` shows the new version while `dotnet restore` still reports `NU1102 ... Nearest
  version: <the previous one>`. Wait, then `dotnet restore --no-http-cache`. It is propagation, not a bad pin.

## What it caught first (2026-11-15)

Written in sebgod/chess while getting it onto an iPad, which repinned most of the train; its first run, afterwards,
found what the repins had missed:

- **WebGl.Renderer 1.38 was built against DIR.Lib 10.4** while chess loaded it with 11.61, across the 11.0 major.
  Chess.Web compiled and its tests passed. WebGl.Renderer 1.39 rebuilt it against 11.62 with no source change.
- **Console.Lib 5.12 was built against DIR.Lib 11.40**, 21 minors behind (additive, so harmless, but nothing had said so).
- **SdlVulkan.Renderer was on DIR.Lib 11.57** while chess was on 11.59.
