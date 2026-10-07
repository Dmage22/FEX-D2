# Diablo II: Resurrected on GameNative: test data

Measurements from test runs on one phone, collected while we look for the cause of D2R's freezes and stutter. These are data points, not a fix. If you run your own tests, you can compare against these numbers.

## Device and base setup

| | |
|---|---|
| Phone | Galaxy Z Fold 8 (SM-F971U), Snapdragon 8 Elite Gen 5, Adreno 840, 12 GB RAM (11.35 GB usable), One UI 9 / Android 17, not rooted |
| FEXCore | `fexcore-2610-d2rfix26` (this fork) |
| Proton | `proton-11.0-2-arm64ec-6` |
| Graphics driver | Wrapper with Turnip T30 (@Mr_Purple_666) |
| DX12 layer | VKD3D-Proton ARM64X (official CI build), BCn emulation `none` |
| Game | 1280x720 container, all D2R graphics settings on lowest, GameNative fps limiter 35 |

## How we measured

Everything was read from outside the game over adb, every 10 seconds: Android's memory figures for the D2R process, swap, memory pressure (PSI), the GPU driver's own counters in `/sys/class/kgsl/kgsl-3d0` (clock, clock limit, load, hang counter), CPU clock and clock limit per core group, and the phone's thermal sensors. VKD3D errors came from `VKD3D_LOG_FILE`. Nothing was attached to or injected into the game.

## Runs

| Run (2026) | Length | What changed | How it ended | GPU hangs* |
|---|---|---|---|---|
| 6 Oct 23:06 | 7 min | baseline, VKD3D 472989a, BCn auto | freeze, `VK_ERROR_DEVICE_LOST` | – |
| 6 Oct 23:19 | 8 min | none (first run with the recorder on) | freeze, `DEVICE_LOST` | +28 |
| 6 Oct 23:33 | 14 min | BCn emulation `none` | freeze, `DEVICE_LOST` | +56 |
| 6 Oct 23:56 | short | Qualcomm system driver (842.19) instead of Turnip | flashing black boxes, stopped | – |
| 7 Oct 00:02 | 17 min | `DXVK_CONFIG=dxgi.maxDeviceMemory=2048` | Android killed GameNative (out of memory) | +22 |
| 7 Oct 00:29 | 10 min | VKD3D master 2230755 (2026-10-06), phone cold at start | D2R closed, memory exhausted, no `DEVICE_LOST` | +4 |
| 7 Oct 05:59 | 18 min | after reboot, ~900 MB of background apps disabled, render scale 86% | freeze with no VKD3D error; all game threads went idle and the GPU dropped to 0% (looks like a deadlock, not a GPU hang) | +5 |
| 7 Oct 06:32 | 30 min | same as previous run | `DEVICE_LOST` | +17 |
| 7 Oct ~09:00 | 0 | Qualcomm driver 891.7 (Adreno 8xx, from Honor firmware) | D2R crashes at launch, with and without BCn emulation | – |
| 7 Oct | ? | Turnip Gen8 V37 instead of T30, anon caching on (not recorded) | crash | – |
| 7 Oct | 30 min | Turnip Gen8 V37, `FEX_DISKCACHEANONCACHING=0` (not recorded) | no crash, but only 20–25 fps instead of 30–35 | – |
| 7 Oct | 20 min | Turnip Gen8 V37, anon caching on, **SMC checks `none`**, idle in town | no problem | 0 |
| 7 Oct | ~60 min | Turnip Gen8 V37, anon caching on, SMC checks `none`, exploring many maps, fights | no freeze, two short dips | 0 |

\* Increase of the kgsl `gpufaults` counter during the run. These are GPU hangs the driver detected; most recover (felt as a stutter), the last one before a freeze does not. The counter resets on reboot.

## Memory

D2R's memory, 30-minute run, at its highest point (07:01, a minute before the crash). The peak column is the highest value of that part across all recorded runs; parts peak at different moments, so that column does not add up.

| Part | 30-min run | Peak, all runs |
|---|---|---|
| Graphics (GPU memory, cannot be swapped) | 4.46 GB | 4.51 GB |
| Game data pushed to swap | 2.67 GB | 4.73 GB |
| Game data in RAM, code, other | 1.70 GB | 2.13 GB |
| **Total (PSS)** | **8.84 GB** | **9.16 GB** |

- D2R's memory keeps growing during play (about 3.8 GB right after loading, 5+ GB of process memory after 15–30 min). Free RAM stays at 0.5–0.9 GB the whole time and Android closes every background app it can.
- Stutter and GPU hang bursts line up with memory pressure spikes (PSI "full" 3–5%).
- These did **not** reduce graphics memory: textures on lowest (always were), BCn emulation `none` (3.9 → 3.7 GB), `dxgi.maxDeviceMemory=2048` (no change), Qualcomm driver instead of Turnip (3.7 GB, plus broken textures).

## Clock speeds and throttling

Samsung limits the clocks from the moment the game loads, not only when the phone is hot.

| | Hardware max | Limit during the 30-min run |
|---|---|---|
| GPU | 1300 MHz | mostly 282–422 MHz, briefly 646 |
| CPU fast cores (6–7) | 4742 MHz | mostly 1500–2700 MHz |
| CPU efficiency cores (0–5) | 3628 MHz | mostly 1200–2700 MHz |

- Even idle in the menu at 36 °C the GPU limit was only 826 MHz; it dropped to 222–342 MHz within about 20 s of loading into the game.
- The actual clock sits on the limit most of the time, and the GPU reads 80–98% busy at that reduced clock.
- Android's casing (SKIN) thresholds on this phone: level 1 at 38 °C, 2 at 40, 3 at 42, 4 at 45, 5 at 47. The phone sits at 42–46 °C during play. Changing "throttle earlier" in Samsung's Thermal Guardian did not change these Android thresholds.
- GameNative is registered in Android's game mode, so Samsung's game service (GOS) manages its limits.

## Current best result

Turnip Gen8 V37 + anon caching on + SMC checks `none`: about an hour of normal play across many maps with no freeze and **no GPU hangs at all** (the kgsl counter for D2R did not move across all V37 runs). One long run so far; the counter-check (V37 with `mtrack`) is still to do.

Two freeze types showed up in the data: GPU hangs ending in `DEVICE_LOST` (every T30 run), and freezes with no VKD3D error where all game threads go idle, which looks like a deadlock. SMC `none` turns off FEX's code invalidation, the path where this fork already found and fixed deadlocks.

SMC `none` means FEX does not notice if code is rewritten in memory. It does not change game memory, but if the game or its protector rewrites code during play, stale translations could cause crashes.

## What looked better (earlier runs)

The two longest runs (18 and 30 min) came after a reboot, with background apps disabled and the newer VKD3D build:

- far fewer GPU hangs: +5 and +17, against +22 to +56 in the shorter runs before
- less game data pushed to swap

The same setup still produced an 18-minute run and a 30-minute run, so a single long run is not proof.

## Ruled out or no effect

- BCn emulation setting (Turnip handles D2R's compressed textures natively; `none` works)
- Reported video memory cap (`dxgi.maxDeviceMemory`)
- FEX translation cost: well under 1% of one CPU thread during play
- Heat alone: one run started already at thermal level 3 and had no hangs for the first 7 minutes

## Open questions

- Whether FEX anon code caching (on by default) plays a role in the freezes. One 30-minute run with it off had no crash but lost about 10 fps; needs repeats before it means anything.
- Why D2R needs about 4.5 GB of GPU memory at the lowest settings and 720p, and whether VKD3D or Turnip allocates more than the game uses.
- Whether active cooling (fan or clip-on cooler) raises the clock limit enough to stop the hangs.
- Whether a different driver changes hang count or memory. Qualcomm 842.19 (system) gives broken textures with BCn `none`; Qualcomm 891.7 does not start D2R at all. Turnip Gen8 V37 (StevenMXZ, mesa-unified gen8 branch) runs with no visible difference from T30, but has crashed too.
- The new VKD3D asks the driver for fault details on `DEVICE_LOST`; Turnip does not provide them, so we still do not know what the GPU is doing when it hangs.
