# H2Bench

Windows FPS benchmark agent: frame timing, an on-screen overlay, local video evidence, and
hardware measurements. Results can be published to the leaderboard on
[h2bench.com](https://h2bench.com).

**This repository distributes the software. The source code is not published here.** What is
published is what a person using the agent or reporting a problem with it needs: release notes,
the measurement methodology, what has actually been verified, and the third-party notices that
cover the redistributed binaries.

## Download

Every build is on the [Releases page](https://github.com/h2bench/h2bench/releases). Two forms per
version:

- **`H2HBench-<version>-Setup-x64.exe`** — installer. Carries the .NET and Windows App SDK
  runtimes plus PresentMon and FFmpeg. Installs a Start menu entry, offers a desktop shortcut,
  and is removed through Windows *Installed apps*. Your benchmark history and videos survive an
  update or an uninstall.
- **`H2HBench-<version>-win-x64.zip`** — portable. Extract the whole archive and run
  `H2HBench.exe`. Keep the `tools` folder next to the executable.

Compare the download against the checksum before running it:

```powershell
Get-FileHash .\H2HBench-<version>-win-x64.zip -Algorithm SHA256
```

The expected SHA-256 is shown on the release page and on the download page on the website.

### The installer is not code-signed

Windows SmartScreen will warn on the installer, and Smart App Control on some systems refuses to
launch it with `ERROR_SYSTEM_INTEGRITY_POLICY_VIOLATION`. This is expected and is not evidence of
a modified file. Until releases are signed, the SHA-256 above is the integrity guarantee.

The agent asks for administrator rights at launch. The elevation is what lets it read CPU clock
and package temperature; without it those two readings show as `—` and everything else works.

## Requirements

- Windows 10 version 2004 or later, or Windows 11, x64.
- Video capture needs a working H.264 encoder: the graphics driver, or the Media Feature Pack.
- Games in exclusive fullscreen need to be returned to the foreground during the countdown.

## Reading the numbers

[docs/METHODOLOGY.md](docs/METHODOLOGY.md) is the reference. In short:

- FPS is measured from PresentMon, as the interval between the application's `Present` calls. That
  is not the same as the interval between photons, so a frame-time figure is often the more
  honest description of a stutter.
- A *low* is not the worst frame. The 1% low is the mean frame rate across the slowest 1% of
  frames, which is why it sits below the average and above the minimum.
- Frame generation cannot be published in this profile. Generated and displayed frame rates need a
  different methodology, and the agent refuses to upload a result with it enabled.
- The video is captured independently of the frame data. The frame numbers are not derived from
  the video.

Russian versions of both documents are kept alongside them, unaltered, in
[docs/ru](docs/ru).

[docs/VALIDATION.md](docs/VALIDATION.md) records hardware and game coverage, and what has and has
not been re-verified after recent changes. It is deliberately candid about the gaps.

## The live overlay

The FPS Monitor shows frame rate, frame time and a pacing graph over a running game, and records
nothing. Start it from the dashboard, or press **Ctrl + Shift + O** from any window: with the
monitor off the combination starts it for the game that is running, and with the monitor already
on it hides and shows the card without stopping the measurement. Settings → Overlay → Hotkey
reports the combination, and reports the reason if Windows refused to register it.

## Reporting a problem

Use [Issues](https://github.com/h2bench/h2bench/issues). The bug form asks for the details that
decide whether a report can be acted on: agent version, OS build, GPU and driver.

Please do not attach a recorded video unless asked. A run's video is tens of megabytes and
contains whatever was on screen. Describe what happened and we will request a capture if we need
one.

## Third-party components

Releases redistribute PresentMon and FFmpeg. Their licences and notices apply to the binaries in
every release and must not be removed. See [THIRD-PARTY.md](THIRD-PARTY.md).
