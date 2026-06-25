# Porting "Modern Matrix" to Windows (.scr)

A self-contained blueprint for re-implementing the macOS Metal screensaver as a
Windows screensaver. This does **not** require reading the Swift source — but the
macOS source in this repo (`Sources/Core`, `Resources/Shaders.metal`) is the
authoritative reference if anything here is ambiguous.

> Workflow reminder: this repo is the bridge between the Mac and Windows machines.
> On Windows, `git clone` it and build the port in a new `windows/` folder.

---

## 0. The shared core — link it, don't re-implement

The **simulation, settings model + derived values, glyph encodings, and world constants
live in portable C99** at `core/mmcore.c` / `core/mmcore.h`. The macOS build compiles that
one file and calls into it (bridged into Swift); **the Windows port must do the same** so a
behaviour tweak is made once and both platforms pick it up.

- **Compile `core/mmcore.c`** into the `.scr` (MSVC `cl` or clang — plain C99, no deps).
- **Include `core/mmcore.h`** and drive the rain through its API:
  - `mm_settings_default()` + `mm_strip_count / mm_fall_speed / mm_mutation_rate` — the model.
  - `mm_sim_create / _update / _advance / _write_instances / _destroy` — the simulation.
    `mm_sim_write_instances` fills an array of `MMGlyphInstance` (8 floats:
    `px,py,pz,cell,bright,cr,cg,cb`) ready to upload as your D3D instance buffer.
  - `mm_encoding_codepoints(enc, out, cap)` — the Unicode code points to rasterise into the
    DirectWrite atlas (cell index == array index).
  - `mm_world()` — the world volume + slot count the camera frames.

So the Windows port implements **only**: the D3D11 renderer, the DirectWrite glyph atlas,
the config dialog + registry/JSON persistence, and the `.scr` host shell. It does **not**
re-implement the rain. **Sections 4, 6, and 7 below now document what `mmcore` already does**
(reference / sanity-check), not something to build from scratch.

---

## 1. What we're building

The 3D "digital rain" from The Matrix: columns of green katakana/digits falling in
a perspective 3D field, bright near-white leader glyph per column, fading green
trail, glyphs occasionally mutating, depth fog, optional camera drift, bloom glow.

It must ship as a **`.scr`** (a normal Win32 PE `.exe` renamed `.scr`) that the
Windows Screen Saver settings can run.

## 2. Recommended stack

- **C++20 + Direct3D 11 + HLSL.** Closest to our Metal design, best perf, full
  control of the `.scr` entry points. (Alternative: C# + Vortice.Windows/Silk.NET;
  easier config UI, more deps. Only consider a shared cross-platform core — Rust+wgpu
  or sokol_gfx — if we later decide to unify both platforms.)
- **DirectWrite + Direct2D** (or GDI+) to rasterize the glyph atlas once at startup.
- Persistence in the **registry** (`HKCU\Software\Chewie\ModernMatrix`) or a JSON in
  `%APPDATA%\ModernMatrix\settings.json`. No sandbox on Windows — sharing config with
  a config dialog is trivial (the painful macOS container saga does NOT apply).

## 3. `.scr` conventions (the host contract)

A screensaver `.exe`/`.scr` parses argv:

| Arg | Meaning |
| --- | --- |
| `/s` | **Run** full-screen (the actual screensaver). |
| `/p <HWND>` | **Preview** — render as a child of the given parent HWND (the little box). |
| `/c` or `/c:<HWND>` | **Configure** — show the modal settings dialog. |
| (none / double-click) | Show the config dialog. |

Rules:
- In `/s`: cover the **virtual screen** (all monitors) with a borderless topmost
  window (or one window per monitor). Hide the cursor.
- **Exit on any input**: record the initial mouse position; on `WM_MOUSEMOVE` past a
  small threshold, `WM_KEYDOWN`, or mouse button → quit. (Preview mode does NOT exit
  on input.)
- Install: user right-clicks the `.scr` → Install, or drop it in `C:\Windows\SysWOW64`.
  The settings UI is still the classic dialog (`control desk.cpl,,@screensaver`).

## 4. The simulation (implemented in `core/mmcore.c` — reference)

> Already implemented in the shared core (`mm_sim_*`). Link `core/mmcore.c` and call it;
> the values below document its behaviour so you can verify the renderer against it.

World volume (units): `halfWidth = 38`, `topY = 28`, `bottomY = -28`,
`spacing = 1.55`, `depthNear = 10`, `depthFar = -42`.
`slotCount = floor((topY - bottomY) / spacing) + 2`  (≈ 38 vertical glyph slots).

**Settings → derived values** (`lerp(a,b,t) = a + (b-a)*t`, all sliders are 0..1):
- `stripCount = round(lerp(60, 900, density))`  — number of falling columns.
- `fallSpeed  = lerp(1.5, 34, speed)`  — world-units/sec a head falls (1.5 = a slow drip).
- `mutationRate = lerp(1.5, 7, speed)` — glyph flips/sec per column.
- Always render at the display's **native refresh rate** (the user-facing frame-rate cap was removed — it only affected smoothness, not the look).

**Per column** (spawn): `x ∈ [-halfWidth, halfWidth]`, `z ∈ [depthFar, depthNear]`,
`speedVariation ∈ [0.7, 1.35]`, `length ∈ [9, slotCount]`,
`yOff ∈ [0, spacing)` (breaks inter-column row alignment — important),
`wavePhase ∈ [0, 2π)`, `cells[slotCount]` = random glyph indices in `[0, glyphCount)`.
Initialize `headY` **above** the top with a randomized offset, e.g.
`headY = topY + random(0, slotCount) * spacing`, so the screensaver starts on a **black
screen** and columns fall in from the top and cascade — do NOT pre-fill the field.

**Per frame** `advance(dt)`:
- `headY -= fallSpeed * speedVariation * dt`  ← read `fallSpeed` **live** every frame
  (do NOT bake absolute speed at spawn — that was a real bug; store the variation and
  multiply by the current `fallSpeed`).
- `headSlot = floor((topY - yOff - headY) / spacing)`.
- Respawn the column when `headSlot - length > slotCount`.
- With probability `mutationRate*dt`, set a random lit cell to a new random glyph.

**Emit instances** (one textured quad per visible glyph): for each column, for
`slot` in `[max(0, headSlot-length+1) .. min(slotCount-1, headSlot)]`:
- `d = headSlot - slot`  (0 = the head)
- `worldY = topY - yOff - slot*spacing`,  position `(x, worldY, z)`
- if `d == 0`: `brightness = 1.55`, color `(0.78, 1.0, 0.82)` (white-green leader)
- else: `t = 1 - d/length`, `brightness = t*t*1.15`, color `(0.10, 1.0, 0.26)`
- if waves on: `brightness *= 0.55 + 0.45*sin(wavePhase + slot*0.5 - time*2.4)`
- glyph cell = `cells[slot]`

Counts: up to ~900 columns × ~30 glyphs ≈ 27k instanced quads — trivial for D3D11
instancing. Rebuild the instance buffer each frame (map/discard).

## 5. Rendering

- **Camera** (right-handed; D3D uses [0,1] depth like Metal): eye `(0,0,48)`,
  center `(0,0,-8)`, up `(0,1,0)`, fovy `46°`, near `1`, far `240`. Aspect = viewport.
  Eye and centre share Y → a **level** view (no downward pitch / "top tilt"); depth comes
  from the Z spread of the rain, not a camera tilt. (Panning, when enabled, drifts between
  the `PanController` presets in `Renderer.swift`, some of which raise/offset the eye.)
- **Billboards**: each instance is a camera-facing quad. In the vertex shader build the
  quad from the camera **right** and **up** vectors: `world = pos + right*corner.x*glyphHalf + up*corner.y*glyphHalf`, `glyphHalf = 0.70`.
- **Glyph atlas UV**: atlas is `cols=16` × `rows` cells; `col = cell % cols`,
  `row = cell / cols`; uv = `(col,row + cellUV) / (cols,rows)`.
- **Fragment**: `coverage = atlas.Sample(uv).r` (single-channel); `rgb = color * brightness * fog * coverage`; output premultiplied. Wireframe mode = draw cell-edge outline instead of sampling.
- **Fog**: distance from camera; `fog = 1 - smoothstep(46, 112, dist)` (fade far glyphs).
- **Blend**: premultiplied-over on a **black** clear — `Src=ONE, Dst=INV_SRC_ALPHA`.
- **Panning** (optional): drift the camera between ~6 preset (eye,center) views with
  smoothstep easing, ~9s per segment. (See `PanController` in `Renderer.swift`.)
- **Bloom** (optional): render scene to an HDR target (`R16G16B16A16_FLOAT`), bright-pass
  (knee ≈ 0.72), 2× separable Gaussian at half-res, composite `scene + blur*intensity*1.6`.
- **HDR** (optional): on Windows use an `R16G16B16A16_FLOAT` swapchain + scRGB / HDR10
  if the display supports it; otherwise clamp. (Lower priority than getting the core right.)

HLSL shaders map 1:1 from `Resources/Shaders.metal` (glyph_vertex/fragment, the
fullscreen bloom passes). Keep the instance/uniform struct layouts identical in spirit.

## 6. Glyph atlas + encodings

> Get the code points from `mm_encoding_codepoints()` in the shared core — the lists below
> are the same data, documented for reference. Rasterise them in order (cell index == index).

Rasterize the glyph set once into a single-channel (R8) texture, `cols=16`, ~72px
cells, **mirrored horizontally** (the film look), with mipmaps. On Windows use
DirectWrite to draw each glyph centered in its cell (font with katakana, e.g.
"Yu Gothic" / "MS Gothic" / "Meiryo"). Cell index == glyph index.

Encodings (the "Matrix encoding" popup):
- **Matrix** (default): half-width katakana `U+FF66..U+FF9D` + digits `0-9`.
- **Binary**: `0 1`
- **Hexadecimal**: `0-9 A-F`
- **Decimal**: `0-9`
- **DNA**: `A C G T`
- **Unicode katakana**: `U+30A1..U+30F6` + half-width katakana + digits.

## 7. Settings (same model as macOS)

> The model + derived values are `MMSettings` / `mm_settings_default()` / `mm_strip_count`
> etc. in the shared core. Your config dialog just reads/writes these fields + persistence.

`density, speed` (0..1 sliders), `encoding` (enum above), and toggles:
`fog, waves, panning, textured, wireframe, showFPS, bloom (+ bloomIntensity), hdr`.
Defaults: density 0.42, speed 0.08 (slow), encoding Matrix, fog/waves/textured/
bloom/hdr on, panning/wireframe/showFPS off, bloomIntensity 0.53.

Config dialog (`/c`): a Win32 dialog (or WinForms/WPF if C#) with the same sliders,
the encoding dropdown, and the checkboxes; persist to the registry/JSON on OK.
(This is where the blog + GitHub links go for roadmap item #5.)

## 8. Frame driving

Drive with a high-resolution timer / `QueryPerformanceCounter`-based loop synced to
the swapchain (vsync via `Present(1,...)` or DXGI flip model). Compute `dt` from one
monotonic clock and clamp to ≤0.1s. (The macOS dual-driver/`dt` saga is Mac-specific;
on Windows a single render loop is clean.)

## 9. Packaging / distribution

- Output `ModernMatrix.scr`. Optionally a tiny installer (Inno Setup / NSIS) or just
  "right-click → Install".
- GitHub release: ship the `.scr` (and the macOS `.saver` + app) as release assets.
- Code-sign if possible (avoids SmartScreen warnings); not required to function.

## 10. Definition of done (parity with macOS)

Falling 3D rain · white-green leaders · fading green trails · glyph mutation · all 6
encodings · density/speed · fog · waves · panning · textured/wireframe ·
FPS overlay · bloom · (HDR if feasible) · config dialog persists · runs `/s` + `/p` +
`/c` correctly · exits on input · multi-monitor.
