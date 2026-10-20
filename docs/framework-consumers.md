# Who builds on the framework

The SharpAstro UI and host libraries (DIR.Lib, SdlVulkan.Renderer, Console.Lib, AppShell, LAN.Lib, Node.Lib,
WebGl.Renderer, Fonts.Lib, SharpAstro.Png) have consumers outside this org as well as in it. A change to one of them is a change to
every app below. This page lists those apps, so a library change gets built and checked against all of them, not
only the one that asked for it.

## The stack

What each library builds on, as its project references say:

| Library | Builds on |
| --- | --- |
| SharpAstro.Png (Codecs) | nothing of the stack |
| Fonts.Lib | SharpAstro.Png |
| DIR.Lib | Fonts.Lib |
| Console.Lib | DIR.Lib |
| WebGl.Renderer | DIR.Lib |
| AppShell | nothing of the stack (pure managed, SDL-free) |
| SdlVulkan.Renderer | DIR.Lib, AppShell, SharpAstro.Png |
| LAN.Lib | nothing of the stack |
| Node.Lib (and Node.Lib.Hosting) | nothing of the stack (the per-user node's socket, lock and Kestrel hosting; .github#5) |

## The consumers

| App | Repo | Uses |
| --- | --- | --- |
| tianwen | SharpAstro/tianwen | DIR.Lib, SdlVulkan.Renderer, Console.Lib, AppShell, LAN.Lib, Node.Lib (tianwen#1703), WebGl.Renderer, SharpAstro.Png |
| pilot v2 | SharpAstro/pilot (`src/Pilot.Gui`, `src/Pilot.Tui`, `src/Pilot.Node`) | DIR.Lib, SdlVulkan.Renderer, Console.Lib, AppShell, LAN.Lib, Node.Lib, SharpAstro.Png |
| FC.SDK viewer | SharpAstro/FC.SDK (`src/FC.SDK.Viewer`) | DIR.Lib, SdlVulkan.Renderer, AppShell |
| LALR.CC TUI | SharpAstro/LALR.CC (`Tui`) | DIR.Lib, Console.Lib |
| chess | sebgod/chess | DIR.Lib, SdlVulkan.Renderer, Console.Lib, AppShell, LAN.Lib, WebGl.Renderer, SharpAstro.Png |
| filetreewalker | sebgod/filetreewalker | DIR.Lib, SdlVulkan.Renderer, Console.Lib |
| YtPlayer | sebgod/YtPlayer | SdlVulkan.Renderer |
| periodic-table-viewer | sebgod/periodic-table-viewer | DIR.Lib, Console.Lib |
| testwinai | sebgod/testwinai | Console.Lib |
| defcon1, dicewars, mathpuzzles | local only, not on GitHub | DIR.Lib, SdlVulkan.Renderer (and Console.Lib for the last two) |

Found by scanning every local clone's project files for the libraries' package ids and `UseLocal*` switches
(2026-10-19). An app that starts using the stack belongs here.

## One source for every consumer

The sebgod apps sit in their own root, `~/source/repos/sebgod/`. That root's library folders are **junctions** to the
clones in this org root (`sebgod/DIR.Lib` → `sharpastro/DIR.Lib`, likewise AppShell, Codecs, Console.Lib, Fonts.Lib,
LALR.CC, LAN.Lib, SdlVulkan.Renderer, StbImageSharp, WebGl.Renderer). Their `..\..\DIR.Lib\…` project references
therefore compile against the same source as tianwen and pilot.

An edit to a library reaches every consumer's next local build at once, with no pull and no version bump, as the
`UseLocal*` switches already do inside the org (`dotnet-ci-pattern.md`). That is the reason to build the consumers
before calling a library change done.

## Where the work is tracked

- **A change one library owns** is an issue in that library's repo (`ShutdownDrain` →
  SharpAstro/SdlVulkan.Renderer#140).
- **Work across repos, or with no owner yet**, is an issue in this repo (SharpAstro/.github) labelled `framework`.
  It links the owning issues and the consumers' (`sebgod/chess#12` links across owners).
- **Code copied from one consumer into another** (tianwen → pilot so far) gets an upstream issue in the library that
  should own it, and the copy names that issue so it can be deleted once the move lands. The home is decided by
  dependency:
  - SDL-free shell plumbing → AppShell.
  - Anything touching `SdlEventLoop` or the window → SdlVulkan.Renderer.
  - Widgets and layout → DIR.Lib.
  - Cells and terminals → Console.Lib.
