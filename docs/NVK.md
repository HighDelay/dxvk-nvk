# DXVK with native NVK on macOS

This branch is based on DXVK **v3.1.1** and repairs startup crashes when a Vulkan
provider does not support Windows shared-memory or shared-semaphore handles.
It is used by [YAAGL NVK](https://github.com/HighDelay/yaagl-nvk) with the native
runtime from [HighDelay's NVIDIA driver](https://github.com/HighDelay/nvidia-macos-driver).

```text
Windows D3D11 game → DXVK → Wine Vulkan → native NVK → NVIDIA GPU
```

The runtime handles macOS surfaces and presentation. This DXVK fork still
translates D3D11/shaders to Vulkan/SPIR-V; it does not replace Wine or implement
macOS presentation itself. The native route bypasses MoltenVK/NVMTL shader
translation, but no controlled performance gain has been measured.

## Repairs

- An image sharing request can be rejected while DXVK falls back to local image
  storage. The requested external handle type was still passed to the memory
  allocator, which called an unavailable `vkGetMemoryWin32HandleKHR`. Keep the
  default no-sharing allocation handle type unless the image is actually shared.
- Optional shared fences/keyed mutexes could similarly reach an unavailable
  semaphore exporter. Refuse shared fence creation early when Win32 semaphore
  sharing is unsupported; the texture's existing local fallback remains usable.

The guards leave supported sharing on its existing path. They do not advertise
fake sharing, return usable external handles for local storage, or alter the
ordinary texture/presentation path. General cross-process sharing remains
unsupported by the current native NVK runtime.

## Build and use

Use the ordinary DXVK Meson build with an x64 MinGW cross compiler and recursive
submodules. Build **both** `d3d11.dll` and `dxgi.dll` from this checkout. The tested
build used llvm-mingw 20261020, glslang 16.6.0, Meson 1.12.1 and Ninja 1.13.2:

```sh
git submodule update --init --recursive
meson setup /absolute/dxvk-build . \
  --cross-file /absolute/win64-cross-file.txt --buildtype release -Ddebug=true \
  -Denable_d3d8=false -Denable_d3d9=false -Denable_d3d10=false
ninja -C /absolute/dxvk-build
```

Keep the pair in the private Wine/NVK package. Its launcher validates the DLL
hashes and installs them into its dedicated prefix with native overrides. Replacing
arbitrary system DLLs or installing this fork alone does not provide native NVK.
Use the companion runtime/launcher setup, preserving its prefix rollback journal.
Release archives contain the matched **x64** `d3d11.dll` and `dxgi.dll`, checksums,
the zlib license and build/source records. Download them from this fork's
[releases](https://github.com/HighDelay/dxvk-nvk/releases), not the ordinary upstream
release, for this repaired pair. The complete runtime/YAAGL package already includes
both DLLs. Do not replace just one DLL or edit the runtime's checksum manifest.

YAAGL prefers the driver's `/Library/GPUBundles/nvmtl/wine` provider while retaining
its compatible bundled Wine. Use `dxvk.enableDescriptorBuffer = False` on this
runtime: descriptor sets restored the invisible ZZZ characters and FPS text.

The ZZZ profile reached menu/login using the existing YAAGL Wine 11.0-1 build,
with its Steam Patch and timeout-fix settings enabled. No Wine binary-code or
macOS kernel changes were needed for these DXVK repairs.

## Tests

Tested on Intel macOS Tahoe 26.7.1, RTX 2060 SUPER:

| Check | Result |
|---|---|
| Original DXVK 3.1.1 shared-texture reproducer | Reproduces execute-at-null fault, matching the game dump |
| Repaired shared-texture probe | Pass: feature levels 11.0/11.1, ordinary/shared/keyed/NT flags, eight GPU pixel readbacks; external-handle requests return errors |
| Ordinary D3D11 regression | Pass: 300 presents, 600 textured draws, depth rejection, alpha blending, 11 readbacks and resize |
| Actual ZZZ process and world rendering | User confirms scenery, characters, UI and FPS overlay; loaded-library maps confirm private NVK and the repaired DXVK pair |
| Sustained ZZZ gameplay / performance comparison | Not qualified |
| Genshin Impact / Honkai: Star Rail | Launcher wiring exists; gameplay untested |

These checks establish the unsupported-sharing repair on this runtime. They do
not qualify every game, real cross-process sharing, other GPUs or Wine builds.
The repaired cases do not execute the supported Windows sharing path, so that
path has only a source-retention check, not a fresh hardware test here.
