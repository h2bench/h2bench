# Agent 2.0 validation

## Presentation layer migration to WinUI 3 — 2026-09-25

- The production WPF UI was replaced with WinUI 3 / Windows App SDK 2.5.1. `H2HBench.Core`,
  PresentMon, WGC/FFmpeg, DXGI/D3D12, the `result.json` and `frames.dat` formats, DPAPI, the mutex
  and the API flows are unchanged.
- `H2HBench.slnx`: Release build succeeds, 0 errors. Core runner: all 79 checks passed, no
  production POSTs.
- Single-file self-contained smoke test: `ExitCode=0`, WMI hardware scan and DispatcherQueue
  confirmed. A visual pass over the Dashboard was done at 125% DPI, checking centring, clipping,
  the compact Quick/Live/Last Result columns and the sidebar.
- On this machine Smart App Control blocks new unsigned managed binaries
  (`ERROR_SYSTEM_INTEGRITY_POLICY_VIOLATION`). The policy was not changed; it was confirmed that
  the single-file executable does launch. A production installer must be signed with a trusted
  certificate.
- The real ETW/game matrix was not re-run after the presentation-layer change. The measurements
  below refer to the WPF UI and the native benchmark backend, which the migration did not alter.

Verified 2026-09-22: Windows 11 build 26200, i5-13450HX 10C/16T, RTX 3050 6GB Laptop, 16 GB DDR5,
Micron SSD, SDK 10.0.401. Release build and self-contained win-x64 publish succeeded. NuGet restore
with `--locked-mode` against the official feed completed with no warnings; compiler warnings 0,
errors 0.

## Automated checks

55 checks of the console regression runner passed: FPS/tail/percentile formulas and invalid
values, swap chains, the real PresentMon 2.3.1 / QPC format, the state machine, consent,
allowlist/privacy, FG and partial upload rejection, stable installation ID, traversal rejection,
RAW, cancel/recovery, 600000 samples, the legacy POST mock, hashes, CPU cancellation, synthetic
units/nulls, challenge token storage, lost chunk response/resume, frozen finish bytes, import
unverified, and legacy ownership proof. No production POSTs were made. Log:
`artifacts/regression-final.txt`.

## Real capture and load

PresentMon was run with ETW rights. A 60-second test of the agent's **own animated WPF window**,
not a game benchmark: 8531 frame samples, coverage 59.94278 s, mean 142.319 FPS. These values
exercise the collector, not game performance.

WGC + NVENC simultaneously: H264 1920×1080, 30 fps, 64.133333 s (including preparation and
countdown), 1924 encoded frames, 7428919 bytes, no audio track. The 8 Mbps target is a target VBR
bitrate; a static window produces a smaller actual file. FFprobe confirmed the format, and the full
SHA-256 is 7f043ee1a963e8dc8750ae3c10fd54051393f17f627f2fa3892d15d10896531e. The MP4 frame was
checked visually: FPS, frame time, settings, UTC, agent 2.0, test ID and the LOCAL marker are all
composited in. These are composited pixels, not a separate overlay window.

Quick and Full synthetic runs were executed for real. Full ran 3 passes with 0 errors; maximum CV
4.3689%. CPU single 15.463M ops/s, multi 213.653M ops/s; RAM 12069.2 MB/s and 64.22 ns; SSD
3243.78 MB/s read, 17187 read / 63820 write IOPS; DX12 graphics 2138.75 draws/s and compute
720.90 GOPS. These are metrics of the new methodology, not an H2H Score, and not a comparison with
the previous agent.

Real NVIDIA load, clock, temperature, power and VRAM were observed, as were system CPU and RAM. CPU
package clock and temperature were not available on this configuration, and unknown readings were
not substituted in. An invalid zero package power reading was excluded. DXGI VRAM was 5.86035 GiB
rather than a truncated 4 GB.

Materials: `artifacts/qa-final/capture-smoke.json`, `synthetic-smoke.json`, `video-probe.json`,
`video-frame.png`, `capture-video.mp4`. Diagnostic data is not included in the user-facing portable
build.

## Limits of this validation

- What was tested is a WPF window and the agent's own offscreen D3D12 workload. The full game
  matrix — DX11/DX12, fullscreen/borderless/windowed, anti-cheat, DPI and multi-monitor, and a
  10-minute continuous real game — has not been run. The 10-minute dataset was verified
  automatically.
- NVENC worked for real. AMF, QSV and MF software are implemented with a runtime probe but have
  not been tested on the corresponding hardware.
- The recording overhead on game scenes is not calibrated. Comparing recording on and off needs a
  repeatable game scenario and a series of runs.
- The future API was tested against a mock HTTP server, including a dropped response.
  Production end-to-end JSON + MP4 waits for the site to implement API-SITE-V2-RU.md; the client
  does not report verified in advance.
- The update loader checks HTTPS, version, size and hash, but a production update has not been
  performed: an older 1.0.0 is published on the site. There is no installer signed with an H2Bench
  certificate; distribution is the portable package.

## 2.0.1 fix validation

- All 56 regression checks passed, including the countdown → video start ordering.
- An ETW check without rights returned ETW_ACCESS_DENIED_DETECTED before recording began.
- The elevated preflight opened and closed the ETW session successfully. A repeat 10-second FPS +
  video run completed: 1400 PresentMon frames, 331 MP4 frames, 1276952 bytes, video complete.
- In the first elevated run the FPS part also completed (1365 frames), but the MP4 was interrupted
  separately. The video error did not reproduce; a bounded recording-detail log was added for
  later diagnosis. The cause of that specific video failure was not established.
- CS2 was closed by the time of this check, so the game was not re-run after the fix. The cause of
  the original FPS failure was established from four saved CS2 result.json files: PresentMon code 6
  and ETW permissions.

## 2.1.0 checks

70 local checks: formulas and lifecycle, the raw file, API mocks, resumable frame transfer on a
lost response, hashes, safe links, separate moderation, unknown fields, and GPU matching. No real
POSTs to the site were made.

A real D3D12 workload was verified on an NVIDIA GeForce RTX 3050 6GB Laptop GPU: the selected
NVIDIA adapter was reported as matched, while the Intel UHD adapter in the same process was a
mismatch. This check does not guarantee compatibility with every game or anti-cheat; unavailable
counters yield unknown.

The full new overlay layout passed a 60-second H.264 1280x720 30 FPS recording with 955 refreshes
and no access errors. FFprobe confirmed the codec, duration and frame rate. The WPF renderings of
both local and server modes were checked. An early intermediate DLL was blocked by Smart App
Control; the final test build was launched without changing any protection settings.

The server-side JS validator was separately checked for exact uint64 comparison, primary chain
selection, a corrupted SHA-256, NaN, an incomplete record, and a frame count mismatch. The server
endpoints, profile binding and file acceptance must be implemented by the site owner per
API-SITE-2.1.md.
