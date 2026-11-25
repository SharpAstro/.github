# iOS apps on a device, from a Windows desktop

For any app on the stack with an iOS head: sebgod/chess's `Chess.iOS` today, TianWen next. The app side (the
`SdlVulkanIosApp` host, the SDL3 and MoltenVK frameworks the package adds, the default log) is in SdlVulkan.Renderer's
README, section "iOS". This page is the device side: getting a build onto an iPad without an Apple Developer
membership, and getting logs and crash reports back off it. Everything here was found the hard way on the desktop in
November 2026; it is written down so the next app does not find it again.

## The build

An iOS build needs macOS, so CI makes it: a macOS job builds `ios-arm64` with signing off and uploads an unsigned
`.ipa` (chess: the `ios` job, artifact `chess-ios-ipa`). Fetch it into a new, empty folder:

```bash
gh run download <run id> -R <owner/repo> -n <artifact> -D <new folder>
```

The `.ipa` is unsigned on purpose: AltStore signs it on install with the tester's own Apple ID.

## The Windows host, once

- **iTunes and iCloud from Apple's website, not the Microsoft Store ones** (`winget install Apple.iTunes`,
  `Apple.iCloud`, the classic iCloud). AltServer talks to their services.
- **Apple Mobile Device Support on its own** (`winget install Apple.AppleMobileDeviceSupport`). The winget iTunes did
  not add it, and without it AltServer sees no device at all.
- **AltServer 1.8.4 or later** (`winget install RileyTestut.AltServer`, then update from its own menu). 1.8.0 fails
  the Apple sign-in with error -80046 since Apple changed its anisette check on 2026-10-22.
- **Firewall: AltServer and Bonjour allowed on the Private profile, and the home network set to Private.** Bonjour
  shipped allowed on Public only, which hid AltServer from the iPad ("AltServer could not be found").
- A Windows popup that LSA protection blocked `mdnsNSP.dll` is harmless. Never turn LSA protection off for it.
- **pymobiledevice3**, for files and logs: `pip install --user pymobiledevice3`, run as `python -m pymobiledevice3`.

## The iPad, once

1. AltServer's tray icon > **Install AltStore** > the iPad. Sign in with a **personal** Apple ID (a free one is
   enough, never a work account). The owner types the password; a session never asks for it.
2. Settings > Privacy & Security > **Developer Mode** on (iPadOS 16+ asks after the first sideloaded install).
3. Settings > General > VPN & Device Management > the Apple ID > **Trust**.

A free Apple ID's limits: a signature lasts **7 days**, **3** sideloaded apps may be active at once (AltStore is one),
and **10** new app ids may be registered a week. AltStore appends the team id to every bundle id (`org.sebgod.chess`
becomes `org.sebgod.chess.<TEAMID>`); `python -m pymobiledevice3 apps list` shows the real one.

## USB or Wi-Fi

USB always works. Over Wi-Fi the iPad has to:

- have Wi-Fi connections enabled for this PC: iTunes' "Sync with this iPad over Wi-Fi", or
  `python -m pymobiledevice3 lockdown wifi-connections on` once while it is on USB;
- be on the same network as the PC (not a guest network);
- be **awake and unlocked**. Asleep, it stops announcing itself, and every command says "Device is not connected".
  This looks like a broken setup and is not.

`python -m pymobiledevice3 usbmux list` shows what the PC can reach (`ConnectionType` `USB` or `Network`). Over Wi-Fi,
give app commands the device's `--udid` (its `UniqueDeviceID` there).

## Installing a build

Keep the bundle id: the install then replaces the app in place and keeps its data (saved games, settings).

1. Copy the `.ipa` into the app's own Documents folder, which the Files app shows as On My iPad > (app):
   ```bash
   python -m pymobiledevice3 apps push --udid <udid> <bundle id> <local .ipa> Documents/<name>.ipa
   ```
   This needs `UIFileSharingEnabled` in the app's `Info.plist`, which the log below needs anyway. AirDrop or iCloud
   Drive into Files works too.
2. AltStore > My Apps > **+** > On My iPad > (app) > the `.ipa`. The picker may open on another folder first, with
   older `.ipa` files in it; browse to the app's own. Over Wi-Fi it takes a minute or two (AltStore re-signs, sends
   the whole app to AltServer and back). If it stalls, plug in USB; do not cancel.

## Logs and crash reports

- **The app's log.** SdlVulkan.Renderer's iOS host (7.109 on) writes its diagnostics to `Documents/<title>.log`,
  replaced on each launch, its first line the app version, device model and OS version. The Files app shows it, or
  from the PC:
  ```bash
  python -m pymobiledevice3 apps pull --udid <udid> <bundle id> Documents/<app>.log <local file>
  ```
- **The device log is no substitute**: it redacts the app's own text as `<private>`.
- **A native crash** leaves no log line. Settings > Privacy & Security > Analytics & Improvements > Analytics Data,
  the newest `<App>-....ips`, shared to iCloud Drive or AirDropped to the PC.

## When the 7 days ran out

The apps stay installed with their data but will not open, and iOS deletes their expired profiles. **Do not delete
the apps**: that loses the data.

1. With the iPad on USB: AltServer > **Install AltStore** over the old one.
2. On the iPad: trust the Apple ID again under Settings > General > VPN & Device Management. A new certificate means
   "Untrusted Developer" once more.
3. AltStore > My Apps > **Refresh All**. Each app is re-signed from the copy AltStore keeps, data intact.

`python -m pymobiledevice3 provision list` shows every profile on the device with its expiry, which is how to tell
"expired" from "broken". To avoid it: open AltStore and Refresh All every few days; its background refresh over
Wi-Fi needs AltServer running and is not reliable on iPadOS.
