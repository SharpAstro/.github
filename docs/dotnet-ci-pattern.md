# The SharpAstro .NET library pattern

Every publishable .NET repo in the `sharpastro` org is built the same way. The
canonical, most-evolved implementation is **`Lzip.Lib`** — read its
`.github/workflows/dotnet.yml` and `Directory.Build.props` when adding a repo or
bringing an old one up to standard.

## Version flows one way

This is the load-bearing convention; the dependency-currency check in `deps.sh`
only works because of it.

1. The repo declares `MAJOR.MINOR` **once**, in the root `Directory.Build.props`:

   ```xml
   <VersionMajorMinor>1.1</VersionMajorMinor>
   <VersionPrefix Condition="'$(VersionPrefix)' == ''">$(VersionMajorMinor).0</VersionPrefix>
   ```

2. CI **reads it back** rather than restating it, and appends the run number as
   the patch component:

   ```yaml
   MM="$(dotnet msbuild Directory.Build.props -getProperty:VersionMajorMinor -nologo)"
   case "$MM" in [0-9]*.[0-9]*) ;; *) echo "::error::..."; exit 1 ;; esac
   echo "VERSION_PREFIX=${MM}.${{ github.run_number }}" >> "$GITHUB_ENV"
   ```

   Published version is `MAJOR.MINOR.<run_number><run_attempt>+<sha>`. Because
   stdout is *captured* here, `DOTNET_NOLOGO: 1` is required at workflow level and
   the shape check must stay — without it a renamed property silently publishes
   everything as `.<run>`.

3. Consumers pin `MAJOR.MINOR.*` and float the patch:

   ```xml
   <PackageVersion Include="DIR.Lib" Version="7.7.*" />
   ```

So a consumer picks up every new producer build for free, but **only within the
minor it pinned**. A producer minor bump therefore has a second half: every consumer
has to bump its pin, or it stays on the old minor indefinitely.

That second half is normally just *not done yet* — the producer bump lands, the
minor gets published, and the consumer pins follow. `deps.sh` calls it
`minor-pending` and `worklist.sh` lists it in dependency order precisely so the
remaining pushes are visible rather than forgotten. It is not a defect on its own.

A pin a full **major** behind is a different thing and worth flagging: nothing on
the producer's current line can resolve, so it is a decision that has quietly
outlived whoever made it.

Release bumps: `MAJOR.MINOR` in `Directory.Build.props`, one line. `1.0 -> 1.1`
for additive features, `1.x -> 2.x` for breaking API changes.

## Workflow skeleton

`.github/workflows/dotnet.yml`, two jobs:

- **build** — `ubuntu-latest`; checkout, `setup-dotnet` (`10.0.x`), resolve version
  prefix, `dotnet restore`, `dotnet build -c Release --no-restore -p:Version=... -p:FileVersion=... -p:ContinuousIntegrationBuild=true`,
  `dotnet test`, upload `**/*.nupkg` as the `nuget-packages` artifact (5-day retention).
- **publish-nuget** — `needs: build`, gated `if: github.ref == 'refs/heads/main' && github.event_name == 'push'`;
  downloads the artifact and runs
  `dotnet nuget push **/*.nupkg -s nuget.org -k ${{ secrets.NUGET_API_KEY }} --skip-duplicate`.

`NUGET_API_KEY` is an **org-level** secret; a new repo must be added to its
"Repositories with access" list or the publish job fails at the push step.

Publishing is artifact-mediated on purpose: packages are built once and pushed
unchanged, so what is tested is exactly what ships.

## Local sibling checkouts override packages

Eight repos (`Console.Lib`, `DIR.Lib`, `FC.SDK`, `Fonts.Lib`, `LALR.CC`,
`SdlVulkan.Renderer`, `WebGl.Renderer`, `tianwen`) carry `UseLocal*` switches that
self-enable when a sibling clone is present:

```xml
<UseLocalLalrCc Condition="'$(UseLocalLalrCc)' == '' AND Exists('..\..\..\LALR.CC\LALR.CC\LALR.CC.csproj')">true</UseLocalLalrCc>
```

This makes the org-root layout load-bearing for builds, not just for git identity
— and it is why stale pins hide. With the sibling cloned you build against its
*source*, so a `3.1.*` pin that is four minors behind never shows up locally. It
only bites in CI and for outside consumers restoring from nuget.org. Treat
`local-src` in the `deps.sh` output as "this edge is untested locally", and set
`-c Release -p:UseLocalLalrCc=false` to reproduce what CI actually restores.

**Both halves of that command matter.** Dropping the switch tests nothing; dropping
`-c Release` tests the wrong thing, in a way that reads as a broken package. A published
package is built Release, so its `#if DEBUG` types are not in it — the debug-inspector
layer (`DIR.Lib.Diagnostics.DebugInspectorCore` and the `ConsoleDebugInspector` /
`DebugInspector` classes that consume it) is entirely gated that way. A **Debug** consumer
restoring that package therefore fails on `CS0234: the namespace 'Diagnostics' does not
exist in the namespace 'DIR.Lib'`, while the identical source builds fine against the
sibling. The message names a namespace, not a configuration, so the obvious conclusions —
the publish dropped files, the pin resolved something old — are both wrong. Same command,
`-c Release`, and it compiles.

Switches in use: `UseLocalConsoleLib`, `UseLocalDirLib`, `UseLocalFontsLib`,
`UseLocalLalrCc`, `UseLocalShapingLib`, `UseLocalSharpAstroCodecs`,
`UseLocalSharpAstroPng`.

**Two path depths, easy to mix up.** A project at `<repo>/src/<proj>/` has
`..\..\..` = the org root, so it reaches a sibling as `..\..\..\DIR.Lib\src\...`. A
project one level shallower at `<repo>/<proj>/` — `LALR.CC/Tui` is the case in the
org — has `..\..\..` = `~/source/repos`, so it must name the org directory too:
`..\..\..\sharpastro\DIR.Lib\src\...`. Copying the wrong prefix produces a switch
whose `Exists()` never fires; it then fails silently by falling back to the package,
which is the failure mode hardest to notice because the build still succeeds.

## Known package-id collisions

Package ids are a global namespace, so two local project names collide with other
publishers on nuget.org. `deps.sh` labels these `foreign-id` by checking the
nuspec `<repository url>`; do not report them as version drift:

- `Canon.EDSDK` (repo `canon-edsdk`) — nuget.org's is authored by **Canon**.
- `FreeTypeSharp.UWP` (repo `FreeTypeBindings`) — belongs to **ryancheung**, the
  upstream project this repo forks. The fork ships as `SharpAstro.FreeTypeBindings`.

## SourceLink comes from the SDK, not a package

**Do not add `Microsoft.SourceLink.GitHub` to a repo here.** The .NET SDK has shipped SourceLink
in-box since .NET 8 and turns it on for any git checkout with a recognised host, so a
`PackageReference` buys nothing and costs the one thing the SDK's copy cannot: it is a RESTORED
package, so it is audited, pinnable, and free to go stale. That is exactly what happened on
2026-09-08, when CVE-2026-62900 (GHSA-23fw-v26w-5fgq) landed against `Microsoft.Build.Tasks.Git`,
which SourceLink depends on. Thirteen repos warned NU1902 on every build, spread across six pinned
versions (8.0.0, 10.0.200, 10.0.201, 10.0.203, 10.0.300, 10.0.301), while the four that had never
referenced it (`AppShell`, `LAN.Lib`, `Lzip.Lib`, `SER.Lib`) were untouched and needed no change.
All thirteen dropped the reference rather than bumping it.

What actually produces a deterministic, source-linked package is the list below, and the package is
not on it:

| setting | where | what it does |
|---|---|---|
| `ContinuousIntegrationBuild=true` | the workflow's build line | rewrites every source path to a `/_/` root, which is what "deterministic" means to NuGet Package Explorer |
| `EmbedUntrackedSources` | csproj | generated sources travel inside the PDB |
| `DebugType embedded` | csproj | the PDB rides in the DLL, so there is no separate symbol package to publish |
| `PublishRepositoryUrl` | csproj | takes the repository URL and branch from git instead of a hand-written `RepositoryUrl` |

Verify a package rather than assuming, by building it the way CI does and reading the result back:

```bash
GITHUB_ACTIONS=true dotnet build <proj> -c Release   -p:ContinuousIntegrationBuild=true -p:DebugType=portable -o out
```

The PDB must carry the document mapping and the nuspec the commit:

```
{"documents":{"/_/*":"https://raw.githubusercontent.com/SharpAstro/<repo>/<sha>/*"}}
<repository type="git" url="https://github.com/SharpAstro/<repo>.git" commit="<sha>" />
```

A repo with a lock file (`FITS.Lib`) needs `dotnet restore --force-evaluate` in the same commit:
locked-mode restore on CI fails NU1004 on a lock that still lists a reference the project dropped.

## Conformance checklist for a repo

- [ ] `<VersionMajorMinor>` in root `Directory.Build.props`, and CI reads it via
      `-getProperty:VersionMajorMinor` (not a hardcoded `VERSION_PREFIX:`)
- [ ] `DOTNET_NOLOGO: 1` in workflow `env`, plus the version shape check
- [ ] `setup-dotnet` at `10.0.x`; action majors consistent with the rest of the org
- [ ] `Directory.Packages.props` (central package management)
- [ ] no `Microsoft.SourceLink.GitHub` reference (the SDK supplies it), and
      `-p:ContinuousIntegrationBuild=true` on the CI build line
- [ ] `.slnx` rather than `.sln`
- [ ] a `dotnet test` step (several publishing repos still have none)
- [ ] publish gated on the default branch + `push`, pushing to `-s nuget.org`
- [ ] every in-org `PackageReference` pinned to the producer's current `MAJOR.MINOR.*`
      (a minor behind is fine mid-release; a major behind needs a decision)

## Only packable projects propagate

An edge held by a non-packable project — a demo `Exe`, a test project, a sample —
cannot reach any downstream consumer, so its pin matters only to that project.
`LALR.CC` is the case to know: its `Console.Lib` and `DIR.Lib` references live in
`Tui/LALR.CC.Tui.csproj`, which is `OutputType=Exe` and does not pack. Only
`LALR.CC/LALR.CC.csproj` ships. That is why the apparent `Console.Lib` ↔ `LALR.CC` ↔
`DIR.Lib` package cycle is not a real one, and why `worklist.sh` builds its publish
order from shipped edges alone.

Watch for the third case too: with central package management a
`<PackageVersion>` entry can outlive the last `<PackageReference>` that used it. No
csproj consumes it, so the version is decorative — delete the line rather than
bumping it. `deps.sh` marks these `unused`. Two live examples: `DIR.Lib` pins
`SharpAstro.Png` that nothing references, and `Codecs` still declares `NUnit` +
`NUnit3TestAdapter` for a vendored `StbImageSharp.Tests` whose csproj is gone (only
stale `bin`/`obj` remain — its real tests are xunit.v3). Always check for a
`<PackageReference>` before calling a pin out of date.

## Drift axes worth watching

Default branch is inconsistent — most repos are on `main`, but `Codecs`,
`FITS.Lib`, `LALR.CC` and `openphd2-multiscope` are on `master`, and the publish
gate is branch-literal, so it must match the repo's actual default.

`LALR.CC` is a deliberate exception: it publishes on **tag** (`v*`) with a guard
step failing the job when the tag disagrees with the csproj `<Version>`, rather
than on every push to the default branch.
