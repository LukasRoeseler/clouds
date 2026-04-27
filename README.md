# Cloude — Procedural Cloud Atlas (WMO)

A single-file 3D web app that procedurally generates and visualizes all 10 WMO
cloud genera in a stylized coastal-city environment. Built for aviation enthusiasts,
pilots in training, and anyone curious about what's actually up there.

Created with [Claude](https://claude.com/claude-code).

![Cloude — coastal scene with cumulus across the sky](docs/screenshot.png)

---

## What it does

- **All 10 WMO cloud genera** rendered as procedural sprite fields:
  Cirrus, Cirrocumulus, Cirrostratus, Altocumulus, Altostratus, Nimbostratus,
  Stratocumulus, Stratus, Cumulus, Cumulonimbus.
- **Sky filled to the horizon** — clouds are scattered across a 3.5km-radius
  field, not just one formation in the middle.
- **Day-night cycle** with realistic sun arc, sunrise/sunset color palette,
  moonlight at night, stars, and city window glow that ramps on after dusk.
- **Lit clouds** — each cloud particle samples its sun-facing factor and tints
  warm on the sun side, cool/grey on the shadow side. At sunset, cloud tops
  glow orange while their undersides go purple-grey.
- **LA-style coastal environment**: tessellated terrain with procedurally baked
  ground texture (street grid, parks, asphalt, beach, chaparral, mountain rock,
  snow caps), instanced lowpoly city, forest belt, distant island silhouettes,
  palms on the beach.
- **Two airplanes** circling the scene at realistic cruise altitudes — a small
  GA aircraft at low cruise (~1-3 km) and a commercial jet near the tropopause.
- **Pilot data overlay** — each cloud type shows flight level range, ISA cloud
  top temperature, composition (water/ice/mixed phase), icing/turbulence/
  visibility hazards, and a brief operational tip.
- **Cloud Quiz minigame** — 20-round identification trainer with the same
  procedural scene as the main view, score tracking, and persistent high score.
- **Aviation-instrument UI** — dark CRT-style panels with amber data readouts,
  Bahnschrift display font, scanlines, and beveled buttons reminiscent of
  early-2000s glass cockpit displays (G1000-style).

---

## Run it

The whole thing is one HTML file with no build step.

```bash
# Local: open it in any modern browser
open index.html

# Or serve it (any static server works):
python3 -m http.server 8000
# → http://localhost:8000
```

To deploy on **GitHub Pages**:

1. Push `index.html` to the root of a repo
2. In Settings → Pages, set source to your default branch
3. Done — it serves as-is

The file pulls Three.js from a CDN (`unpkg.com/three@0.160.0`); no other
dependencies, no API keys, no service workers.

---

## Controls

| Action               | Mouse / Touch                | Keyboard          |
|----------------------|------------------------------|-------------------|
| Orbit camera         | Drag                         | —                 |
| Zoom                 | Scroll wheel / pinch         | —                 |
| Reset camera         | —                            | `Space`           |
| Randomize cloud      | Click ⟳ Randomize            | `R`               |
| Open quiz            | Click ▶ Cloud Quiz           | `Q`               |
| Scrub time of day    | Drag the Time slider         | `←` / `→` (30 min)|
| Close quiz           | × button or click outside    | `Esc`             |

Wind speed, wind direction, and cloud density also have sliders; Randomize
shuffles cloud type plus all atmospheric parameters except time of day.

---

## Project structure

Everything lives in `index.html`. It's organized into numbered sections inside
the module script:

```
1.  WMO cloud database (10 genera, full pilot data)
2.  Procedural noise (value noise + fbm)
3.  Procedural cloud textures (canvas-based puff bake)
4.  Three.js scene + lights + sky shader (gradient + sun/moon disks)
4a. Coastal-city environment (terrain, ocean, city, forest, islands, palms, planes)
4b. Time-of-day system (10-keyframe palette, sun/moon arc, fog, ambient)
5.  Cloud builders
    – buildCloud (single Cb formation)
    – buildSingleClump (one cumulus puff)
    – buildCloudField (horizon-filling field, used for all non-Cb genera)
    – buildScene (genus → single or field)
6.  Camera controls + animation loop
7.  UI bindings (panel, sliders, dropdowns, info card)
8.  UTC clock
9.  Resize handling
10. Cloud Quiz module (20 rounds, persistent high score)
10b. Keyboard shortcuts
10c. Mobile bottom sheet (responsive layout)
11. Kickoff
```

---

## How the visuals work

**Procedural cloud textures** — A 256×256 canvas per cloud type, painted with
fbm value-noise distorted from a radial falloff. Cirrus uses an elongated
horizontal-fiber kernel; dense types (Ns, Cb) use a sharper alpha curve.
Textures are cached and reused across thousands of sprite billboards.

**Cloud fields** — For all genera except Cb, `buildCloudField()` scatters
50–200 sub-cloud "clumps" across a 3500-unit radius using a golden-angle
spiral with seeded jitter. Each clump is itself a small group of sprites built
by `buildSingleClump()` with genus-specific layout (puffy cauliflower, fibrous
streaks, regular grid for Ac/Cc, sheet for As/Cs/Ns, lumps for Sc, low fog-like
patches for St). Per-genus tuning of count, particle density, vertical jitter,
scale boost, and opacity boost makes each type visually distinctive.

**Sky** — A single skybox sphere with a custom GLSL shader does a vertical
gradient (ground → horizon → zenith) plus a sun disk with halo and bloom, and
a moon disk that ramps in at night. All driven by shared uniforms updated by
`setTimeOfDay(hours)`.

**Lighting on clouds** — Each particle gets a baked `litFactor` (0–1) at
build time computed from `position · sunDir` plus height-in-cloud bias. The
time-of-day system mixes `cloudTintColor` (warm sun side) and `cloudShadeColor`
(cool shadow side) per particle and multiplies by a sun-altitude `dayBoost`.
At night, both colors lerp toward a moonlight blue.

**Terrain** — A 220-segment tessellated plane uses a hand-painted 2048×2048
canvas texture mapped 1:1 to terrain bands (beach, asphalt city with street
grid, forest belt, chaparral foothills, snow-capped mountains). Heights come
from `landHeight(x, z)` which combines fbm noise with deterministic band
elevations.

**City + forest** — Both use `THREE.InstancedMesh` (one draw call per material)
to render ~600 buildings and ~700 trees efficiently. Building heights follow
an LA-style distribution: most are 1–5 stories, a downtown cluster has
sparser skyscrapers up to ~310m. Building windows use an emissive material
that fades from invisible by day to glowing warm amber at night.

---

## Cloud data

All altitudes in the `CLOUDS` database are real WMO values in feet, e.g.:

```js
Cu: {
  abbr: 'Cu', name: 'Cumulus', etage: 'Low',
  altMin_ft: 1500, altMax_ft: 18000, vertical: true,
  desc: '...',
  icing: 'mod', turbulence: 'mod', visibility: 'good',
  flightTip: 'Fair-weather Cu (Cu humilis) → light bumps. Cu congestus → strong updrafts...'
}
```

The visual altitude in scene units (`etageBaseHeight`) uses a compressed
bands so all genera fit in the camera's view; the info card always shows the
real flight-level range so the pilot data stays accurate.

---

## Performance

Most heavy lifting is one-time at startup:

- Procedural noise → fbm tables baked into 256×256 cloud textures
- 2048×2048 terrain ground texture baked once on canvas
- City + forest instanced meshes built once
- Sky / sun / moon are uniform-driven, no per-frame geometry rebuilds

Per-frame work is dominated by sprite drift + per-particle lighting updates
when time-of-day changes. On a mid-range laptop the scene runs at ~60 fps; on
an older device, the cloud field count automatically falls back to floor
values that still fill the sky.

---

## Browser support

- Chrome / Edge: full support, best performance
- Firefox: full support
- Safari (desktop + iOS): full support including the bottom-sheet mobile UI,
  notch-safe insets, and pinch-zoom
- Anything that supports WebGL 2 + `<script type="module">` will run it

---

## License

MIT — do whatever you want, just don't claim you built it from scratch.

---

## Credits

- WMO cloud genera reference: WMO International Cloud Atlas
- Aviation hazard guidance is a simplified summary of standard pilot training
  material; **not** a substitute for proper meteorology training or briefing
  before flight
- Built collaboratively with Claude
