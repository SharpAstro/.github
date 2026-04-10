# SharpAstro

A .NET ecosystem for astrophotography automation — from hardware control to automated imaging sessions.

## Core

- **[tianwen](https://github.com/SharpAstro/tianwen)** — Full-featured .NET library for astronomical device management, automated imaging, and image processing. Supports ASCOM, INDI, ZWO, and Meade protocols. Includes an automated session orchestrator (cooling, focusing, guiding, meridian flips, dithering), FITS I/O, star detection, plate solving, and a GPU-accelerated FITS viewer.
- **[TianWen.DAL](https://github.com/SharpAstro/TianWen.DAL)** — Vendor-neutral Device Access Layer defining the core abstractions for consumer astronomical hardware (cameras, focusers, filter wheels). Pure interfaces targeting .NET Standard 2.0.
- **[zwo-sdk-nuget](https://github.com/SharpAstro/zwo-sdk-nuget)** — NuGet package wrapping ZWO's native C SDKs (ASI cameras, EAF focusers, EFW filter wheels) via P/Invoke, with bundled native binaries for Windows and Linux. Implements the `TianWen.DAL` interfaces.

## Utilities

- **[openphd2-multiscope](https://github.com/SharpAstro/openphd2-multiscope)** — Multiplexing proxy for PHD2's JSON-RPC event API, enabling synchronized dithering across a multi-telescope setup.
- **[skyfi-forwarder](https://github.com/SharpAstro/skyfi-forwarder)** — Bridges SkyWatcher SkyFi UDP packets to a serial port, letting Wi-Fi-only software control serial-connected mounts.
- **[FC.SDK](https://github.com/SharpAstro/FC.SDK)** - A replacement for EDSDK.dll that relies entirely on PTP/WPD when available, and also supports using USB directly like libgphoto2 does, but in C#. AOT compatible.
