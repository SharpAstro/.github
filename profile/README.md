# SharpAstro

A .NET ecosystem for astrophotography automation — from hardware control to fully automated imaging sessions — plus a set of general-purpose, AOT-compatible .NET libraries that grew out of that work.

## Astrophotography

### Core

- **[TianWen](https://github.com/SharpAstro/tianwen)** (天文) — Free, open-source astronomical imaging suite for .NET. Manages cameras, mounts, focusers, filter wheels, and guiders via ASCOM, Alpaca, ZWO, QHYCCD, Meade, Skywatcher, OnStep, and iOptron, with first-class support for multi-OTA (dual-rig) setups. Ships as a NuGet library (`TianWen.Lib`), a cross-platform CLI with an interactive TUI, a headless REST API server (used by [Touch N Stars](https://github.com/Touch-N-Stars/Touch-N-Stars)), a standalone FITS viewer, and an integrated N.I.N.A.-style GUI. Includes an automated session orchestrator (cooling, focusing, guiding, meridian flips, dithering), FITS I/O, star detection, and plate solving.
- **[TianWen.DAL](https://github.com/SharpAstro/TianWen.DAL)** — Vendor-neutral Device Access Layer defining the core abstractions for consumer astronomical hardware (cameras, mounts, focusers, filter wheels). Pure interfaces, implemented by the device SDKs below.

### Device SDKs & bindings

- **[ZWOptical.SDK](https://github.com/SharpAstro/ZWOptical.SDK)** — NuGet packages wrapping ZWO's native C SDKs (ASI cameras, EAF focusers, EFW filter wheels) via P/Invoke, with bundled native binaries for Windows and Linux (x86/x64/ARM). Implements `TianWen.DAL`.
- **[QHYCCD.SDK](https://github.com/SharpAstro/QHYCCD.SDK)** — QHYCCD native SDK wrapper for .NET, with `TianWen.DAL` integration.
- **[Canon.EDSDK](https://github.com/SharpAstro/Canon.EDSDK)** — Modern .NET 10 P/Invoke bindings for Canon's EDSDK, fully source-generated (`LibraryImport`) and NativeAOT-compatible.
- **[FC.SDK](https://github.com/SharpAstro/FC.SDK)** ("Free Canon SDK") — Canon EOS camera control via PTP over USB and Wi-Fi, pure managed C# with no EDSDK binary required — an alternative for platforms where Canon's closed-source, Windows-only EDSDK isn't an option.

### Formats & imaging

- **[SER.Lib](https://github.com/SharpAstro/SER.Lib)** — Pure-managed reader/writer for the SER planetary-imaging video format (Lucam Recorder v3), with memory-mapped, frame-accurate random access even in multi-gigabyte files. Consumed by `tianwen`'s FITS viewer to open SER captures as a frame sequence.
- **[FITS.Lib](https://github.com/SharpAstro/FITS.Lib)** — Fork of the CSharpFITS library with a significant performance boost for writing large, multi-dimensional int/short arrays — the common case for astronomical imaging data.

### Utilities

- **[openphd2-multiscope](https://github.com/SharpAstro/openphd2-multiscope)** — Multiplexing proxy for PHD2's JSON-RPC event API, enabling synchronized dithering across a multi-telescope setup.
- **[usbreset](https://github.com/SharpAstro/usbreset)** — Small USB reset tool, handy for recovering wedged cameras/focusers without a physical replug.

## General-purpose .NET libraries

Pure-managed, AOT/trim-friendly libraries with no native dependencies, built to support the astrophotography stack (rendering, terminal UI, codecs) but independently useful.

- **[DIR.Lib](https://github.com/SharpAstro/DIR.Lib)** — Device-independent input and rendering foundation shared by GPU (SDL3 + Vulkan) and terminal (console) front ends.
- **[SdlVulkan.Renderer](https://github.com/SharpAstro/SdlVulkan.Renderer)** — SDL3 + Vulkan rendering surface built on `DIR.Lib` primitives, AOT-compatible.
- **[Console.Lib](https://github.com/SharpAstro/Console.Lib)** — Library for building terminal applications: dock-based layouts, widgets, mouse/keyboard input, VT styling, and Sixel graphics. Also ships `mdcat`, a `cat`-for-Markdown CLI built on it.
- **[Fonts.Lib](https://github.com/SharpAstro/Fonts.Lib)** — Pure-managed OpenType/TrueType font loading and rendering: hinting, CFF outlines, COLR color glyphs, bitmap emoji, variable fonts, PostScript Type 1, and WOFF/WOFF2.
- **[Codecs](https://github.com/SharpAstro/Codecs)** — Family of pure-managed, AOT-compatible image codec NuGets for .NET, with a facade that sniffs a byte stream's magic bytes and dispatches to the right decoder.
- **[Lzip.Lib](https://github.com/SharpAstro/Lzip.Lib)** — Pure-managed lzip (LZMA1) compressor/decompressor, including parallel decoding of multi-member files, compatible with the reference `lzip` binary.
- **[LALR.CC](https://github.com/SharpAstro/LALR.CC)** — LALR(1) parser-table generator and runtime for C#, with a YAML grammar schema and a Roslyn source generator pipeline.
