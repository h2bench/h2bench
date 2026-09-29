# Methodology h2h-present-interval/1.0

Source: PresentMon 2.3.1, `--v1_metrics --qpc_time`, column `msBetweenPresents` (parsed
case-insensitively). These are the intervals between the application's Present calls, which are
not necessarily the intervals between physical screen presentations. Dropped presents are not
discarded. Frame Generation cannot be published under this profile: generated and displayed frame
rates require a separate methodology.

Positive finite intervals are kept. The first zero entry does not participate. Zero, negative, NaN
and Infinity values are not replaced with invented ones. The PID is filtered by both PresentMon and
the parser. The complete set of valid frames is stored along with its swap chain address. For the
final metrics the chain with the most frames is chosen, and on a tie the lowest address. Data from
different swap chains is never summed together. Several chains raise FRAME_CAPTURE_UNSTABLE; a
chain change in the middle of a run can reduce coverage, and such a run should be repeated.

Frame time: `(QPCTime − sessionStartQpc) / Stopwatch.Frequency`. Telemetry uses a Stopwatch
started at the same point, before the capture process is created. Any difference in clock start is
the overhead of a few calls, not the asynchronous delivery time of ETW. A telemetry sample is
stamped with the time its poll began. Intervals after the end of the configured window are not
accepted. Before the final calculation PresentMon is stopped and its output is drained.

For N intervals in milliseconds:

- Duration = sum(frameTime) / 1000 — the frame-covered duration, which may be slightly shorter
  than the selected window.
- Average FPS = N × 1000 / sum(frameTime).
- Median FPS = 1000 / median(frameTime).
- Minimum FPS = 1000 / max(frameTime); Maximum FPS = 1000 / min(frameTime).
- 1% low = 1000 / mean(the slowest ceil(N × 0.01) intervals).
- 0.1% low = 1000 / mean(the slowest ceil(N × 0.001) intervals).
- Percentile P: sort ascending, index `(N − 1) × P`, linear interpolation between neighbours.

Values are not rounded in the stored data. The UI rounds only what it displays. The live panel
uses the most recent 2000 events, not the whole run; the final metrics are recomputed from the
complete set. The history graph is decimated for drawing, RAW is not.

## RAW format v1

Little-endian, 8-byte header: ASCII `H2HF`, int32 version=1. Then 24-byte records: float64
timestampSeconds, float64 frameTimeMs, uint64 swapChainAddress. Up to 2 million events (48 MB plus
managed memory), window up to 600 seconds. On reaching the limit a partial run is saved with an
error. Compact binary without gzip is used so that recovery works after a crash; an incomplete
trailing record is ignored during recovery.

The writer flushes every 250 ms, and telemetry is checkpointed every 5 seconds. Atomic replacement
of result.json/telemetry.json prevents reading half-written JSON. This protects against ordinary
process failures; it is not a guarantee of durability against power loss or disk failure.

## Quality flags

- SHORT_TEST: frame-covered duration below 30 seconds.
- BENCHMARK_INTERRUPTED: cancelled, the game closed, an exception, or recovery after a failure.
- FRAME_CAPTURE_UNSTABLE: no frames, several swap chains, or the timestamp range of the selected
  chain covers less than 85% of the requested window.
- GPU_NOT_FULLY_LOADED: mean available NVIDIA GPU usage below 80%.
- FRAME_GENERATION_ENABLED: the user indicated frame generation; upload is blocked, local
  measurement is allowed.

CPU/GPU workload values of the synthetic profile are not derived from game FPS. Local FPS is not
converted into an H2H Score. The formulas of the new methodology have to be agreed with the owner
of the server ratings before any broad comparison with the previous agent.

## Additional quality flags

HIGH_BACKGROUND_CPU: at least one available background CPU sample above 20% (system CPU minus
process CPU, normalised by logical processors). THERMAL_THROTTLING: an explicitly available sensor
returned true; an unknown value does not mean throttling is absent. VIDEO_CAPTURE_FAILED: the MP4
write or verification failed. SYNTHETIC_COMPONENT_FAILED: one component failed, the remaining
measurements were kept.

## System methodology h2h-synthetic/2.0.0

Quick: 1 pass, 1.5 s per sub-measurement. Full: 3 passes, 3 s. Initialisation and shader
compilation are excluded from the timing window. The result is the mean of the available values;
the passes are stored separately. Variance is the maximum population coefficient of variation
across the metrics, in %. A single pass yields no variance, and errors are not turned into zeros.

- CPU: either 1 or `Environment.ProcessorCount` dedicated threads with a start barrier. One
  operation is 32 iterations of uint xorshift(13,17,5) and multiply 0x9e3779b9. Ops/s. Predominantly
  integer workload.
- RAM bandwidth: Buffer.BlockCopy 64 MiB, bytes/seconds/1e6. Cache and runtime affect the result.
- RAM latency: random dependent pointer chase over 4M int32 (16 MiB), seed 42, ns/access. The
  working set may fit in LLC, so this is not pure DRAM latency.
- SSD: dedicated 64 MiB scratch file, unbuffered + write-through, aligned buffers, DeleteOnClose.
  Sequential read 1 MiB in MB/s; random 4 KiB QD1 read/write in IOPS. Small working set, not a
  sustained SSD workload. User files and the raw device are not touched.
- GPU graphics: D3D12 on the selected DXGI adapter, offscreen 1920×1080 RGBA8 fullscreen triangle,
  fixed 64-iteration pixel shader, batch of 4 draws, fence before completion. Draws/s
  (fps-equivalent), not game FPS.
- GPU compute: 65536 threads, 128 dependent float multiply/add iterations, batch of 4 dispatches
  with UAV barriers, SM5.1 precise, 2 operations/iteration. A GOPS workload, not vendor peak
  TFLOPS. Readback verifies the final result.

## Video

WGC HWND capture, up to 1920×1080 with aspect preserved, H264 30 fps, target 8 Mbps / max 10 Mbps,
no audio, GOP 60. The encoder is chosen by a trial encode of NVENC/AMF/QSV/MF software.
Fragmented MP4 keeps completed fragments on failure. FPS comes only from PresentMon. FFmpeg
drop/duplicate counters describe the encoder pipeline, not dropped game presents.

The overlay is composited with drawtext, refreshes up to 4 times per second, and contains the CPU
and GPU names, the installed amount of RAM, UTC, app version, settings and the run ID. From version
2.0.3 the Code line is shown only when a server nonce is present; the Code local placeholder was
removed from both the screen and video overlays. Without a server it reads LOCAL / NOT SERVER
VERIFIED. From version 2.0.1 recording starts after the countdown, and the elapsed time in the
overlay begins at the FPS window. The start of the MP4 and of the RAW differs. The full SHA-256 of
the MP4 and RAW is recorded after the writers are closed. Client metadata is not a cryptographic
attestation of the game.

## 2.1.0 changes: frame data and the passport

The per-frame file frames.dat is transferred whole through the resumable frames API when the
capability benchmarkFrames=h2h-frames/1 is available. The original file containing all swap chains
is sent; primarySwapChain and primaryFrameCount designate the chain used for the final metrics. The
server is obliged to verify the hash, the binary format and the chain selection itself. The graph
is built from the real records, not from summary FPS; decimation is only acceptable for display.

The game's GPU is checked from the RunningTime increments of the Windows GPU Engine type 3D
counters for its PID, bound to the DXGI LUID of the selected adapter. Several active adapters mean
ambiguous, and the absence of available data means unknown. This check is a heuristic, not
cryptographic proof. A mismatch blocks submission; an unknown adapter is marked
GPU_ADAPTER_UNVERIFIED. User-entered API and V-Sync values have the source user_game_settings. The
DXGI runtime does not distinguish DX11 from DX12, and SyncInterval does not prove the actual V-Sync
mode. The game version is the FileVersion of the executable, not a guaranteed Steam build ID.

The screen and video overlays share the OverlayDesign layout: dark panel, teal FPS, separate metric
columns, CPU/GPU names, RAM/VRAM, settings, ID, UTC, duration, agent version, and the real nonce
when present. The video updates individual fields through open fixed-size files with shared
reading; they are not renamed or truncated while FFmpeg reads them. Transferring or acknowledging
the files does not mean the result was accepted by moderation.
