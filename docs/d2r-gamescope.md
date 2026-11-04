# D2R in GameNative's gamescope mode: first-frame hang and fix

*Fork maintainer notes, 2026-11-03. Not part of the FEX project.*

## TL;DR

With **Display server = gamescope** (GameNative nightlies from 2026-11), Diablo II: Resurrected freezes on a black screen right after start-up (FPS ≈ 0, GPU 0 %, game still running underneath). SurfaceFlinger mode works.

**Fix:** add this environment variable to the container. All six entries are needed:

```
VKD3D_DISABLE_EXTENSIONS=VK_KHR_present_wait,VK_KHR_present_wait2,VK_KHR_present_id,VK_KHR_present_id2,VK_KHR_swapchain_maintenance1,VK_EXT_swapchain_maintenance1
```

Also use **GameNative's Proton 11.0-2 rebuild 9** (`11.0-2-arm64ec-9`, the app default since 2026-10-28) or newer. It is built with XFixes (proton-wine#105); older builds (`11.0-2.2`, `.82`, our `-15`) are not, and under gamescope the controller-mode cursor (the circle D2R moves with the stick via `SetCursorPos`) is invisible because Xwayland only shows such warps while Wine hides the cursor through XFixes. Mouse/keyboard mode is unaffected. Verified 2026-11-03.

Newer VKD3D-Proton (3.1+, including our ARM64X builds) triggers the hang; GameNative's default VKD3D 3.0b does not, which is why it went unnoticed. GameNative (utkarsh) is fixing the layer; once that ships, retest without the variable.

The hang's cause is not FEX, Proton, the Steam overlay or D2R's protector. It is a bug in GameNative's closed-source Vulkan layer for gamescope mode. The variable only changes which Vulkan features VKD3D uses; it does not touch the game.

## Cause

In gamescope mode the game's Vulkan goes through `gn-vkshim` (`lib/libadrenotools_gamescope.so` in the gamescope payload, built from GameNative's private `gamenative-drm` project). The shim advertises present-wait and swapchain_maintenance1 present fences to the guest but never completes them:

1. vkd3d-proton (FIFO / VSync path, `libs/vkd3d/swapchain.c`) presents frame 1 with a present ID and a present fence.
2. Its `vkd3d-swapchain-sync` thread calls `vkWaitForPresentKHR(id 1, UINT64_MAX)`.
3. The shim hands the frame to gamescope but never reports it as presented (`presented=0`) and never gets the buffer back (`rel=0`). Wayland only releases a buffer once a newer one replaces it.
4. vkd3d's frame-latency fence is never signalled, so the game's main thread blocks before frame 2. Deadlock on the very first frame.

Evidence:
- `GNDRM_LOG_VKSHIM=1` (hidden shim log switch): one acquire, present and commit (`dpy_err=0`), then `rel=0 presented=0 slots=[0:-F 1:-- 2:-- 3:--]` and repeated "present worker has no work".
- adb `/proc` per-thread sampling while hung: `vkd3d-swapchain-sync` and the main thread asleep with zero wakeups; `gn-present` and the gamescope compositor idle.

## What did not work

| Attempt | Result |
|---|---|
| ReShade effect (forces composition) | flip "Invalid argument" errors gone, still hangs; the flip errors are a side effect |
| Frame cap off | still hangs |
| `VKD3D_SWAPCHAIN_PRESENT_MODE=MAILBOX` | still hangs (probably ignored: the shim appears to offer only FIFO) |
| D2R in-game VSync off | still hangs |
| `GN_VKSHIM_NO_PROXY=1` | D2R refuses to start |
| Hiding only `present_wait`/`present_id` (4 entries) | game runs with sound but shows nothing: the maintenance1 present fences are unsignalled too |
| Our Proton vs GameNative's Proton | no presentation-related difference; Proton is not involved |

## Where the code lives

| Payload file | Source |
|---|---|
| `bin/gamescope`, `gamescopereaper` | [GameNative/gamescope](https://github.com/GameNative/gamescope) (public; patch series in `android/patches/gamescope/`) |
| `bin/Xwayland` | [GameNative/xserver](https://github.com/GameNative/xserver) (public) |
| `lib/libadrenotools_gamescope.so` (gn-vkshim, **the bug**), `libgndrm_shim.so`, `libgamenative_drm.so` (display daemon) | `gamenative-drm`: private, not buildable by us |

The payload is downloaded from `https://downloads.gamenative.app/gamescope-gamenative-<VERSION>.tzst` (`GamescopePayload.VERSION` in the app). The gamescope command line is built in `BionicProgramLauncherComponent.wrapWithGamescope`.

## Debugging tips for this mode

- gamescope's own log lines (`session-blank`, `selecting mode`, flip errors) appear in the Wine log. `session-blank` is GameNative patch A33 (it blanks the screen until the game's surface commits) and is harmless here.
- `GNDRM_LOG_VKSHIM=1`: verbose shim log (swapchain, acquire, present, commit, release counters). Other shim switches found in the binary: `GN_VKSHIM_NO_PROXY`, `GN_VKSHIM_NO_GPL`, `GN_VKSHIM_NO_ADRENOTOOLS`, `GN_VKSHIM_SYSTEM_DRIVER`, `GN_VKSHIM_PREFER_DRIVER`, `MESA_VK_WSI_PRESENT_MODE`, `WRAPPER_MAX_IMAGE_COUNT`.
- `VKD3D_DEBUG=warn` (without `VKD3D_LOG_FILE`) puts VKD3D messages in the Wine log, e.g. `Extension "VK_KHR_present_wait" is disabled` to confirm the fix took effect. Remove it for normal play.
- For a hang, sample per-thread CPU and wakeups over adb (read-only, nothing attaches to the game): `/proc/<pid>/task/*/stat` and `status` (`voluntary_ctxt_switches`). Don't attach a debugger to D2R (protector).

## Upstream status

Reported to GameNative on 2026-11-03; utkarsh is fixing it in the layer. Suggested fix for them: complete present waits and signal present fences on gamescope's presentation feedback or flip event, or stop advertising present_wait(2), present_id(2) and swapchain_maintenance1 to the guest.
