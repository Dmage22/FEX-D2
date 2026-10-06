> ## This fork: Battle.net games on GameNative (Diablo II: Resurrected, Overwatch 2)
>
> *This section is written by the fork maintainer (Dmage22), not by the FEX project. Everything below it is the unmodified upstream FEX readme.*
>
> This fork adds a few small fixes so Blizzard's protected launchers work under Wine ARM64EC on Android (GameNative).
> They fix emulation compatibility only. **Nothing here touches, hides from or bypasses any anti-cheat.** Use at your own risk.
>
> - **Download:** [Releases](https://github.com/Dmage22/FEX-D2/releases) → `fexcore-2609-d2rfix25.wcp`. Import it in GameNative (Contents) and pick it as the FEXCore version.
>   Don't use the `-debug` builds (extra logging) or `maple` builds (a different game).
> - **Source:** branch [`d2r/clean`](https://github.com/Dmage22/FEX-D2/tree/d2r/clean).
> - **Also needed:** a Proton with the matching Wine fixes, either [GameNative/proton-wine#47](https://github.com/GameNative/proton-wine/pull/47) once merged, or [`proton-11.0-2-arm64ec-6`](https://github.com/Dmage22/proton-wine/releases).
> - **Tested on:** Galaxy Z Fold 8 (Snapdragon 8 Elite Gen 5, Adreno 840, 12 GB RAM), GameNative bionic container, headless Steam.
>
> ### Common FEXCore preset (both games)
>
> | Setting | Value |
> |---|---|
> | TSO (`FEX_TSOENABLED`) | **on** |
> | Vector TSO (`FEX_VECTORTSOENABLED`) | **on** |
> | Memcpy/Set TSO, Half-barrier TSO | off |
> | Multiblock | **on** |
> | SMC checks | `mtrack` |
> | Max inst | 5000 |
> | X87 reduced precision, Host features, Hide hypervisor bit | off |
> | Environment variable | `FEX_DISKCACHE=1` (anon caching is on by default) |
>
> Keep Wine debug channels **off** while playing; they make the games very slow.
>
> ### Diablo II: Resurrected
>
> - Steam mode: **headless Steam** (needed for the Battle.net login).
> - DX wrapper: **VKD3D**, with the "DXVK Version" field set to a GameNative DXVK (e.g. `2.6.1-gplasync`); it provides `dxgi`.
> - Graphics driver: Wrapper + a recent Turnip build.
> - Optional environment: `VKD3D_CONFIG=no_upload_hvv,nodxr`, `MESA_SHADER_CACHE_MAX_SIZE=4096MB`.
> - In game: low textures, frame cap 30 (60 with frame generation).
> - Known issue (under investigation): an occasional GPU freeze ("device lost") after a few minutes of play.
>
> ### Overwatch 2
>
> - DX wrapper: **DXVK**, ideally a native ARM64EC build, e.g. `dxvk-3.1.1-gplasync-arm64ec-dm.wcp` from [Dmage22/dxvk releases](https://github.com/Dmage22/dxvk/releases). A version name containing "async" turns async on automatically.
> - Environment: `DXVK_CONFIG=dxgi.maxDeviceMemory=2048;dxgi.maxSharedMemory=2048;dxvk.enableGraphicsPipelineLibrary=False`
>   (caps the video memory the game sees, and stops DXVK from precompiling thousands of unused shaders).
>   Set it as a container environment variable; GameNative's own DXVK memory option has no effect.
> - Environment: `MESA_SHADER_CACHE_MAX_SIZE=4096MB`.
> - Graphics driver: Wrapper + Turnip. The Qualcomm 849 driver left objects unrendered in our tests.
> - In game: low textures, frame cap 30; optionally FSR 1 with render scale 70–80 %.
> - Expect: fast loading after the first launch and around 30 fps in practice mode. Large multiplayer fights are CPU-bound and can dip on current phones.

---

[中文](https://github.com/FEX-Emu/FEX/blob/main/docs/Readme_CN.md)
# FEX: Emulate x86 Programs on ARM64
FEX allows you to run x86 applications on ARM64 Linux devices, similar to qemu-user and box64.
It offers broad compatibility with both 32-bit and 64-bit binaries, and it can be used alongside Wine/Proton to play Windows games.

It supports forwarding API calls to host system libraries like OpenGL or Vulkan to reduce emulation overhead.
An experimental code cache helps minimize in-game stuttering as much as possible.
Furthermore, a per-app configuration system allows tweaking performance per game, e.g. by skipping costly memory model emulation.
We also provide a user-friendly FEXConfig GUI to explore and change these settings.

## Prerequisites
FEX requires ARMv8.0+ hardware. It has been tested with the following Linux distributions, though others are likely to work as well:

- Arch Linux
- Fedora Linux
- openSUSE
- Ubuntu 22.04/24.04/24.10/25.04

An x86-64 RootFS is required and can be downloaded using our `FEXRootFSFetcher` tool for many distributions.
For other distributions you will need to generate your own RootFS (our [wiki page](https://wiki.fex-emu.com/index.php/Development:Setting_up_RootFS) might help).

## Quick Start
### For Ubuntu 22.04, 24.04, 24.10 and 25.04
Execute the following command in the terminal to install FEX through a PPA.

```sh
curl --silent https://raw.githubusercontent.com/FEX-Emu/FEX/main/Scripts/InstallFEX.py | python3
```

This command will walk you through installing FEX through a PPA, and downloading a RootFS for use with FEX.

### For other Distributions
Follow the guide on the official FEX-Emu Wiki [here](https://wiki.fex-emu.com/index.php/Development:Setting_up_FEX).

### Navigating the Source
See the [Source Outline](docs/SourceOutline.md) for more information.
