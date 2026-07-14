# Modifications

This is a **modified version** of Video2X. It is not an official release by the Video2X authors.

Required by the GNU Affero General Public License v3.0, section 5(a): a modified work must carry
prominent notices stating that it was changed, and the date of the change.

## Who changed it

Fork: <https://github.com/FortunaCournot/video2x>
Maintained for the **VR we are** toolset: <https://github.com/FortunaCournot/comfyui_stereoscopic>

## What was changed

**2026-07-14 — Destroy the ncnn GPU instance before exit instead of during DLL unload.**

Upstream Video2X terminates with a segmentation fault on **every** run, including successful ones. It
prints `Video processed successfully`, writes its full processing summary, and only then dies. A
successful run and a failed one are therefore indistinguishable by exit code, which forces every
script driving Video2X to ignore the exit code and inspect the output file instead.

The cause: nothing destroys ncnn's Vulkan instance on purpose. ncnn creates it lazily and only tears
it down from a static destructor plus an `atexit()` handler it registers itself. On Windows ncnn is a
*shared* library, so both of those run while the loader is already unloading DLLs at process exit —
and the teardown calls into other DLLs (`vkDeviceWaitIdle`, `vkDestroyInstance`,
`glslang::FinalizeProcess`), which is not allowed at that point.

The fix adds `video2x::release_gpu_resources()` and calls it from an RAII guard in `main()`, while
the process is still healthy. This is what ncnn's own reference tools (`rife-ncnn-vulkan`,
`realesrgan-ncnn-vulkan`) do.

Verified by building two Windows Release binaries from this repository's CI that differ *only* in
this change:

| | RIFE | Real-ESRGAN |
|---|---|---|
| unpatched 6.4.0 | exit 139, 139, 139 | exit 139 |
| with the fix | exit 0, 0, 0 | exit 0 |

The produced videos are byte-identical. The fix changes the exit code and nothing else.

**Also included, without which the Windows binary cannot be built at all today:**

- `humbletim/setup-vulkan-sdk` bumped from `v1.2.0` to `v1.2.1`; `v1.2.0` pulls `actions/cache@v2`,
  which GitHub now fails automatically.
- The Vulkan SDK bumped from `1.3.204.0` (2021) to `1.4.313.0`; the old one declares
  `cmake_minimum_required` below 3.5, which CMake 4 refuses.
- The Boost DLL is matched by pattern instead of the hard-coded MSVC toolset `vc143`; current
  runners build it as `vc145`, so `install` failed after everything had already compiled.

## Upstream

All of the above has been offered back upstream: **<https://github.com/k4yt3x/video2x/pull/1500>**

If and when upstream merges it and cuts a release, this fork becomes unnecessary and the VR we are
installer should point back at the official release.

## Source

The Corresponding Source of any binary published from this fork is this repository, at the tag the
release was built from — same server, freely available, no charge (AGPL-3.0 section 6(d)).
