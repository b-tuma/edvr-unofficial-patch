# Handoff: AMD rig, VR star crispness and related ideas (2026-10-18)

## Status

- **State: IDEAS ONLY.** Nothing built, nothing flown, no code changed. Written from a cloud
  session (Linux, no game, no build) for the next session on the user's Windows gaming PC.
- **First job of the next agent: install the build tooling on that PC** (section "Setup"
  below). Claude Code is already installed; everything else is not. Then build, smoke, and
  fly the AMD baseline.
- **The user's rig:** AMD Radeon RX 9070 XT (RDNA 4), Quest 2 (considering a PSVR2),
  Windows. First AMD discrete card known to fly EDVR. No DLSS: the temporal modes available
  are `on` (TAA), `fsr` (FSR 3.1, community D3D11 port) and `xess` (flat only).
- **The user's current settings:** all anti-aliasing off, HMD Quality pushed as high as the
  card allows to hide edge aliasing. So `fix.ui_quality` is inert today (it needs a temporal
  mode on).
- **The complaint:** stars are soft under any temporal AA or upscaler in VR. Flat is crisp.
- **Open hypotheses for the star blur** (none measured):
  - H1: the stars are sub-pixel, so a temporal filter averages them away. The ini already
    says so: "sub-pixel detail such as stars shows at its true fraction of a pixel".
  - H2: wrong motion vectors on sky pixels during rotation.
  - H3: FSR 3.1's known weakness on tiny moving highlights.
  - Discriminator: AA off against `fsr` at HMD Quality >= 1.0, head still against a steady
    yaw.
- **Next flight (the AMD baseline, no code):** `fix.temporal_aa = fsr` at HMD Quality >= 1.0
  (1:1, no upscale) and `fix.ui_quality = 100`, then AA off. Read the GPU census with
  `python tools\edvr_log.py --target steam --expect-build HEAD`. This also closes "flight 4"
  in docs/fsr-upscaler-design-2026-09-16.md (an AMD supporter's log).
- **Ideas, ranked:** (1) sky history mask at 1:1; (2) FSR 4 through the D3D12 bridge;
  (3) a stars layer after the upscale; (4) flat HDR. Each has a section below.
- **Ruled out, do not re-propose:** sharpening as a star fix (AGENTS.md and
  docs/hologram-depth-2026-09-24.md); foveated rendering on this rig (section "Foveation");
  OptiScaler as a drop-in beside EDVR (section "OptiScaler"); HDR in the headset (section
  "HDR").

## Setup on the Windows PC (the next agent does this)

The agent installs these itself, from PowerShell, asking the user to approve the UAC prompts.
Check the winget IDs with `winget search` first; they are believed, not verified here.

1. **Build Tools for Visual Studio 2022**, not 2026, and the IDE is not needed. Install the
   C++ workload with the Windows SDK and the CMake tools:
   `winget install Microsoft.VisualStudio.2022.BuildTools --override "--passive --wait --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended --add Microsoft.VisualStudio.Component.VC.CMake.Project"`.
   Why 2022: `tools\fetch_ffx_dx11.py` hardcodes the CMake generator "Visual Studio 17 2022"
   and looks for VS2022's bundled CMake and dumpbin; CI runs Windows Server 2022.
   `build.bat` finds the toolchain through vswhere (`VC.Tools.x86.x64`), which Build Tools
   satisfies. Not yet confirmed on a Build Tools-only machine; if the FSR fetch misses the
   bundled CMake, `pip install cmake` is its third fallback.
2. **Python 3.12 or newer, on PATH** (`build.bat` runs `where python.exe`):
   `winget install Python.Python.3.12`, then `pip install -r tools\requirements.txt`.
3. **Git for Windows**: `winget install Git.Git`. Claude Code also uses its Git Bash.
4. Open a new shell so PATH picks the installs up.
5. One-time fetches from the repo root: `python tools\fetch_openxr_loader.py` (required) and
   `python tools\fetch_ffx_dx11.py` (FSR 3.1). Skip `fetch_ngx.py` (DLSS, NVIDIA only).
   `fetch_xess.py` is optional (flat only).
6. Build by absolute path: `<repo>\build.bat --wait-lock`. Green writes
   `build\full_build_receipt.json`. Then `build\smoke.exe build\d3d11.dll`.
7. `python tools\install_edvr.py --target steam --dry-run`, then without `--dry-run`.

Expect the build to report DLSS as absent; that is supported (`package.bat --no-dlss`).
Whether it is silent about it is unverified.

## Idea 1: sky history mask at 1:1 (smallest, best fit for this user)

At HMD Quality >= 1.0 the temporal pass runs 1:1 ("no resolution given up", edvr.ini's
`temporal_aa` text). The stars are then already rendered at full size, so they need no
redraw. They only need the filter to take the current frame on sky pixels. The world keeps
its AA, and the UI is already layered by `fix.ui_quality`.

- Mask source: far-plane depth (EDVR already reads the game's depth in every temporal mode),
  or the star draw's own pixels if the stars are a separate draw (unknown, see below).
- FSR 3.1 takes it as the reactive or transparency-and-composition mask. EDVR runs FSR with
  the reactive mask off (open item D3 in the FSR doc). EDVR's own TAA is EDVR's code; its UI
  resolve already treats masked pixels. DLSS preset K ignores such masks
  (docs/dlss-blur-elements-2026-10-14.md, Finding 1), which does not matter on AMD.
- Risks: things over the sky that a depth mask catches (canopy glass, glare, dust, nebulae,
  small distant bodies; edvr.ini's ui_quality text explains why the canopy stays out of the
  UI layer), and a possible halo at ship silhouettes against stars.
- Evidence first: whether the canopy and the glare write depth over sky pixels. Use a pixel
  probe or census on a star and on canopy-covered sky.

## Idea 2: FSR 4 through the D3D12 bridge

FSR 4 (the ML upscaler) is D3D12-only and targets RDNA 4, the user's card. It is "route 2"
in docs/fsr-upscaler-design-2026-09-16.md section 5, parked until "an AMD tester has
reported, and an RDNA3 or RDNA4 tester exists". The user is that tester.

- Plumbing partly exists: the XeSS engine's D3D11-to-D3D12 bridge (`src\d3d11\xess_engine.cpp`,
  docs/xess-flat-intel-b390-2026-10-11.md), modelled on OptiScaler's dx11-with-dx12 path.
- Work: take the bridge from flat to VR (two eyes a frame); an FSR 4 engine beside the XeSS
  one; read the licence of AMD's signed DLLs before shipping them (unverified, per the doc).
- Believed cost: 0.3-1 ms a stereo pair for the cross-device hand-off.
- The FSR baseline flight comes first, so FSR 4 has something to beat.

## Idea 3: stars drawn after the upscale

Only needed if the user ever runs below 1:1. This is the `fix.sharp_weapons` pattern: a
per-eye additive HDR layer composited after the upscale (dlss-blur doc, "Weapons fire").
`fix.ui_quality` already moves the supercruise orbit lines, the supercruise bars and the
space dust into its layer when the layer is wider than the render.

- The key unknown is how Elite draws normal-space stars. Either they are their own draw
  (point sprites; the witchspace starfield is `9AEC596A2B036EA6`, docs/particle-billboards.md),
  which can be redirected directly, or they are baked into the per-system sky cube (1024 to
  4096 per face, docs/black-holes.md "lensing" notes, forum-sourced). A baked cube would
  need a sky resample at output size, capped by the cube's own texel density.
- One capture with the census and pixel probe on a star settles which.

## Idea 4: HDR

- **In the headset: no.** No mainstream PC VR runtime is known to accept HDR swapchains over
  OpenXR; PSVR2 is believed SDR on PC.
- **Flat HDR monitor: feasible but large.** HDR swapchain and colour space; replace Elite's
  SDR tonemap with a display-referred one; composite the UI at paper white (the UI layer
  helps); move the temporal pass before the tonemap, since it runs on the tonemapped image
  today (docs/black-holes.md, B4); keep EDHM working. Try Windows 11 Auto HDR first, no code.

## Foveation: why not

All three past attempts are retired. VRS was NVAPI-only (no AMD D3D11 path) and gave zero
fps (docs/performance.md). Foveated DLSS saved about 1.1 ms of 8.9 and is NVIDIA-only
(docs/foveated-dlss-design-2026-09-14.md). Eye-tracked crops are ruled out under upscaling.
The bound: Elite renders the whole eye itself; EDVR can only cheapen its own passes. And
foveation lowers quality, the opposite of the user's goal.

## OptiScaler

Not a drop-in: it swaps a game's own temporal upscaler calls, and Elite's VR path makes none
(EDVR does the upscaling). It also hooks the same D3D11/DXGI layer. EDVR already uses its
D3D11 FSR 3.1 port (`tools\fetch_ffx_dx11.py`) and its D3D11-on-12 bridge as the XeSS
reference.

## Other AMD-only tech, judged

AGS (D3D11 extensions): small, only if a profile shows EDVR's own passes as the AMD
bottleneck; no VRS. Anti-Lag 2: flat only, minor (VR pacing is the runtime's). Fluid Motion
Frames and frame generation: not for VR. Radeon Super Resolution: nothing to build.

## Layer costs (for reference)

From docs/edvr-gpu-trims-2026-10-14.md: RTX 5090, 2016^2 to 4032^2, hangar, UI quality 100.
HDR HUD seed 0.291, UI layer reissues 0.212, UI layer composite 0.164: about 0.67 ms a frame
for ui_quality, of EDVR's 5.07. Not measured: the game's cost of larger panels, sharp
weapons, ui_quality 125, or anything on AMD. The baseline flight's census answers the AMD
part.
