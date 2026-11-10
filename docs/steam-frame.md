# Steam Frame Runtime

This fork adds Steam Frame VR to the `takedawn` toolkit branch from
[xenophobentx/xboxrecomp](https://github.com/xenophobentx/xboxrecomp).
The game integration lives in
[astromaddie/takedawn](https://github.com/astromaddie/takedawn/tree/steam-frame-vr).
Use `steam-frame-vr` in both repositories. The toolkit builds runtime libraries;
it isn't a standalone game, and it doesn't include an XBE or game assets.

## What Changed

- `XBOXRECOMP_VK_XR=ON` adds the OpenXR path to the Linux Vulkan backend.
  It's off by default and isn't enabled for Android, Windows or macOS.
- OpenXR creates the Vulkan instance and device through
  `XR_KHR_vulkan_enable2`. A headset thread runs the OpenXR frame loop and
  shares the renderer's queue under its lock.
- The Vulkan renderer records the finished picture's passes and draws them
  again for each eye. It transforms perspective geometry and reconstructs
  CPU-projected effects using the game camera's projection.
- The game selects immersive racing or a stereo cinema screen through
  `xbox_vr_scene()`. The HUD has its own head-following panel. Hold View for
  a second to recenter.
- Frame controllers feed the game's Xbox pad input, including rumble.
  Menu is Start, View is Back, and the bumpers are Black and White.
- Linux thread placement uses core speed to put the guest on a big core and
  the GPU command executor on the prime core.

The implementation is in `src/d3d/nv2a_vk_xr.c`, the stereo replay in
`src/d3d/nv2a_pb_vulkan.c`, and `D3D8_GLSL_STEREO` in
`src/d3d/d3d8_vsh_glsl.c`. Takedawn supplies the scene hook and launch defaults.
Other games need their own camera and scene integration; enabling the build
flag doesn't turn every Xbox title into VR.

## Build with Takedawn

The [game's build guide](https://github.com/astromaddie/takedawn/blob/steam-frame-vr/docs/steam-frame.md)
is the complete route. Its Dockerfile builds the dependencies as well as the
game, so you don't need to install its graphics SDKs on the host.

You need Git, Python 3.10 or newer, Docker with Linux ARM64 support, and your
own North America retail Xbox disc files. Apple silicon and Linux ARM64 can
run the container natively. Docker on x86-64 Linux or Windows WSL2 needs
ARM64 emulation; this isn't an x86-64 build or a cross-compiler setup.

```sh
git clone --branch steam-frame-vr https://github.com/astromaddie/takedawn.git
git clone --branch steam-frame-vr https://github.com/astromaddie/xboxrecomp.git
cd takedawn
python3 -m venv .venv
.venv/bin/python3 -m pip install capstone
# Put your own disc files in "Burnout 3 Takedown/", including default.xbe.
SPLIT=250 ./regen.sh
DOCKER_DEFAULT_PLATFORM=linux/arm64 JOBS=2 tools/frame/build.sh all
```

The toolkit must be the sibling `../xboxrecomp` for these commands. On a Mac
using Colima, start the ARM64 VM with `colima start --memory 4 --cpu 4` first.
The script writes `build-frame/bin/burnout3` and stages
`build-frame/package/Takedawn`. It doesn't upload or launch anything.
The generated C and the disc files stay local and must not be published.

The image uses Ubuntu 22.04, clang 15, SDL3 with sdl2-compat, Vulkan headers,
glslang, SPIRV-Tools and OpenXR SDK 1.1.63. The exact dependency versions are
in Takedawn's `tools/frame/Dockerfile`. The build enables ARMv8.2 crypto
instructions for the game's installer. The package carries the needed
user-space libraries; SteamOS supplies Vulkan and the GPU driver.

For your own game integration, CMake needs to find Vulkan and glslang before
it enables `XBOXRECOMP_VK_XR`. It then requires the OpenXR loader's CMake
package (`OpenXR::openxr_loader`). Takedawn's build script supplies the SDK
paths and `-DXBOXRECOMP_VK_XR=ON`. Check the configuration output for both:

```text
xboxrecomp: Vulkan backend on
xboxrecomp: OpenXR on the Vulkan backend on
```

Those lines have additional details in the real log. Passing the XR option
without a discovered Vulkan backend does not enable VR. A successful build
alone also doesn't establish that a title's camera or controller mapping works.

## Run and Check

Use the game's deploy script and start the installed title from Steam, which
supplies the controller routing and per-app runtime settings. The Frame must
be in Developer Mode with SSH key access configured. The game guide covers
the data directory, deployment and settings. No device address is built into
these instructions.

The [recorded Takedawn checks](https://github.com/astromaddie/takedawn/blob/steam-frame-vr/docs/steam-frame.md#measured)
cover the original 2026-10-20/21 runs: the 120 Hz headset loop, 60 Hz game
simulation, per-eye image comparisons and GPU timings at several eye sizes.
These are Takedawn results, not a compatibility claim for other games.
The original game still culls against its own camera, and target-feedback
draws can fall back to flat rendering. See the game guide for the current
limits and the diagnostic scripts; headset tests should only run when the
device is free.
