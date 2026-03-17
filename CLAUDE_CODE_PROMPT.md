# Claude Code Prompt: 3D Atmospheric Layers with Thermospheric Density & Satellite Conjunction

## Context

I have two existing HTML files:

1. **`index.html`** — A Seeze homepage with a Three.js conjunction visualization that already renders:
   - A textured Earth (day/night/clouds/bump via `SphereGeometry` + `MeshPhongMaterial`)
   - 5 atmospheric layer shells (troposphere → exosphere) using custom `ShaderMaterial` with fresnel + edge-fade
   - A density field inside the thermosphere (vertex-colored spheres at 200/400/600 km altitudes with diurnal color animation)
   - Two orbital arcs (objects A & B) converging at a conjunction point with a wireframe "Conjunction Assessment Zone" cube
   - A risk probability field (volumetric glow near the conjunction point)
   - 4 view modes toggled by buttons: Orbit, Conjunction, Risk Field, Atmosphere
   - Uses Three.js r128 + OrbitControls from CDN

2. **`earth_density_viewer.html`** — A standalone thermospheric density viewer with:
   - Manual orbit controls (no OrbitControls dependency)
   - Density shells at 200/400/600 km with vertex coloring from a log₁₀(ρ) color LUT
   - Wind tracer particles (arrow-head InstancedMesh) animated along density gradients
   - Time-animated frames with play/pause, opacity, altitude toggle controls
   - Embedded demo data (6×12×3 lat/lon/alt grid, 24 diurnal frames)

## What I Want Built

Create a **new single-file HTML page** (`atmosphere_full.html`) that combines and extends both visualizations into one unified 3D experience. Specifically:

### 1. Show ALL atmospheric layers clearly
- Render all 5 layers as distinct, labeled, semi-transparent concentric shells around Earth:
  - **Troposphere**: 0–12 km (innerR = 1.0, outerR ≈ 1.002)
  - **Stratosphere**: 12–50 km (outerR ≈ 1.008)
  - **Mesosphere**: 50–80 km (outerR ≈ 1.013)
  - **Thermosphere**: 80–700 km (outerR ≈ 1.11) — this is the active layer
  - **Exosphere**: 700–10,000 km (outerR ≈ 2.57)
- Each layer should have a unique color, fresnel-based edge glow, and boundary ring at each transition
- Layers should be individually toggleable (checkbox UI) and have adjustable opacity
- Use an **exaggeration factor** (slider, default ~5×) so lower layers are visible — the current `index.html` uses `outerR` values like 1.04, 1.09, 1.14, 1.65, 2.4 which are already exaggerated; keep this approach but make it configurable
- Add **CSS2DRenderer labels** (or HTML overlay labels) positioned at each layer showing name + altitude range, auto-hiding when layer is toggled off

### 2. Thermospheric density field (inside the thermosphere shell)
- Port the density visualization from `earth_density_viewer.html`:
  - 3 altitude shells (200 km, 400 km, 600 km) with vertex-colored `SphereGeometry`
  - Color mapped from log₁₀(ρ) using the existing 5-stop LUT (deep purple → blue → cyan → orange → yellow)
  - Time-animated diurnal cycle (24 frames, interpolated)
  - Wind tracer particles using `InstancedMesh` with arrow geometry
- Density shells should render **inside** the thermosphere atmospheric shell, visually contained
- Include the existing controls: play/pause, speed, frame scrubber, opacity, altitude toggles, tracer toggle + scale

### 3. Satellite conjunction visualization (inside the thermosphere)
- Port the conjunction elements from `index.html`:
  - Two orbital arcs (Object A in violet, Object B in cyan) at radius ≈ 1.5 (representing ~3,200 km altitude, within thermosphere range)
  - Satellite markers (small glowing spheres) at the leading edge of each arc
  - Forecast arcs (dashed lines) showing predicted future trajectory toward conjunction point
  - **Conjunction Assessment Zone**: wireframe cube at the convergence point with animated edges
  - **Risk probability field**: volumetric glow/heatmap near conjunction point, color-coded green→yellow→red
  - **TCA label** (Time of Closest Approach) positioned above the conjunction point
- The conjunction should visually sit **within** the thermosphere layer, making it clear these objects are embedded in the density environment

### 4. View modes / camera presets
Provide toggle buttons (like the existing Orbit / Conjunction / Risk Field / Atmosphere):
- **Overview**: Pull back to see all layers, Earth, and orbits
- **Thermosphere Focus**: Zoom into thermosphere, show density + conjunction prominently
- **Conjunction Detail**: Close-up on the conjunction zone with risk field
- **Atmosphere Layers**: Highlight all 5 atmospheric shells with labels
- Smooth animated camera transitions between modes (lerp position + lookAt target over ~1s)

### 5. UI/Controls panel
- Bottom control bar with: play/pause, speed slider, frame scrubber, global opacity
- Side panel or overlay with: layer toggles (checkboxes for each atmosphere layer + density shells), exaggeration factor slider
- Color legend (vertical gradient bar) for density values
- Conjunction info panel: show TCA, miss distance, probability of collision

---

## Library Recommendations & Technical Pointers

### Core 3D Engine
- **Three.js r128+** (already used in both files) — stick with this for consistency
  - CDN: `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`
  - OrbitControls: `https://unpkg.com/three@0.128.0/examples/js/controls/OrbitControls.js`

### For atmospheric layer rendering
- **Custom `ShaderMaterial`** (already used) — fresnel rim lighting + depth-based alpha for each shell
  - Vertex shader: pass world position, normal, view direction
  - Fragment shader: fresnel = `pow(1.0 - dot(N, V), 2.5)`, edge fade via `smoothstep`, additive blending
  - Use `THREE.BackSide` rendering for inner glow effect
  - Set `renderOrder` per layer to control transparency sorting
- **`THREE.TorusGeometry`** for boundary rings between layers (thin, semi-transparent)

### For density field visualization
- **Vertex-colored `SphereGeometry`** with `THREE.MeshBasicMaterial({ vertexColors: true, transparent: true, blending: AdditiveBlending })`
- **`THREE.InstancedMesh`** for wind tracer particles (much more performant than individual meshes for 100s of arrows)
  - Use `THREE.ConeGeometry` + `THREE.CylinderGeometry` merged for arrow shape
  - Update instance matrices each frame based on wind vectors
- **Color LUT**: precompute a 256-entry Float32Array mapping log₁₀(ρ) → RGB, sample with linear interpolation

### For conjunction / orbital mechanics
- **`THREE.Line`** with `THREE.BufferGeometry` for orbital arcs — set vertices from parametric orbit equations
- **`THREE.LineDashedMaterial`** for forecast/predicted trajectory segments
- **`THREE.EdgesGeometry`** + `THREE.LineSegments` for the conjunction assessment zone wireframe cube
- **Custom `ShaderMaterial`** for the risk probability field — volumetric sphere with radial gradient, color mapped green→yellow→red based on collision probability
- Small `THREE.SphereGeometry` with emissive `MeshBasicMaterial` + `THREE.PointLight` for satellite markers (glowing dots)

### For labels and HUD overlays
- **`THREE.CSS2DRenderer`** (`CSS2DObject`) — perfect for text labels that track 3D positions but render as crisp HTML
  - CDN: `https://unpkg.com/three@0.128.0/examples/js/renderers/CSS2DRenderer.js`
  - Attach labels to each atmospheric layer boundary and to conjunction point
- Alternatively, use absolute-positioned HTML `<div>` elements and project 3D positions to screen coords via `vector.project(camera)` (the approach used in `index.html`)

### For smooth camera transitions
- Use `THREE.Vector3.lerp()` to interpolate camera position and lookAt target
- Store camera presets as `{ position: Vector3, target: Vector3 }` objects
- Animate over ~60 frames in the render loop using an easing function

### For post-processing (optional, adds polish)
- **`THREE.EffectComposer`** + **`UnrealBloomPass`** for glow effects on the conjunction zone and satellite markers
  - CDN: `https://unpkg.com/three@0.128.0/examples/js/postprocessing/EffectComposer.js` (+ RenderPass, UnrealBloomPass, ShaderPass)
  - Selective bloom: use `layers` system to only bloom specific objects
- **`THREE.RGBShiftShader`** or chromatic aberration for cinematic feel (subtle)

### For data / physics accuracy (optional)
- **satellite.js** (`https://cdn.jsdelivr.net/npm/satellite.js@4/dist/satellite.min.js`) — for TLE parsing and SGP4 orbit propagation if you want real orbital mechanics
- **NRLMSISE-00** model approximation — the demo data already uses a simplified diurnal density model; for more realism, implement the empirical atmosphere model coefficients

### Performance tips
- Use `InstancedMesh` for anything with many copies (particles, debris)
- Set `frustumCulled = false` on large transparent shells to prevent popping
- Use `depthWrite: false` on all transparent materials
- Set explicit `renderOrder` values: Earth (0) → inner layers (1-5) → density shells (10-12) → orbits (20) → conjunction elements (30)
- Use `requestAnimationFrame` with a delta-time accumulator for frame-rate-independent animation

### Styling
- Dark space background with CSS radial gradients (as in `earth_density_viewer.html`)
- Star field: absolute-positioned small circles with CSS `twinkle` animation
- Control panels: semi-transparent dark backgrounds (`rgba(0,0,0,0.65)`), monospace font, cyan/blue accent colors
- Match the existing Seeze design language: JetBrains Mono font, navy/teal color palette

---

## File Structure
Single HTML file with everything inlined:
- `<style>` block for all CSS
- `<body>` with canvas container, star field, control panels, labels, legend
- `<script>` tags for Three.js + OrbitControls + CSS2DRenderer CDN imports
- Main `<script>` block with all scene setup, data, animation loop

## Reference Files
- Copy the Earth texture loading approach from `index.html` (lines 1016-1033): day map, bump map, specular map, cloud layer
- Copy the atmospheric shell shader from `index.html` (lines 1083-1127)
- Copy the density data generation and vertex coloring from `earth_density_viewer.html` (lines 183-214 for data, lines 309+ for Earth + shells)
- Copy the conjunction geometry from `index.html` (lines 1346-1460 for orbital arcs, satellite markers, conjunction cube)
