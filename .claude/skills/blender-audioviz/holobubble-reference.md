# HoloBubble Pattern — Architecture & Reference

Geometry-agnostic transparent emission pattern system. Each surface object owns its
own properties and material. A single frame handler processes all registered surfaces.

## Design principles

- **Axis-based naming** (`x_*`, `y_*`) instead of ring/spoke — scales to any geometry
- **Per-surface properties** — each object controls its own animation independently
- **Per-axis color** — independent emission intensity and hue per axis per surface
- **Negative rates** — all `*_per_beat` and `*_speed` properties support negative
  values for reverse direction
- **Snap control** — per-axis `x_snap`/`y_snap` (0=smooth slide, 1=instant flip)
- **Two group modes** — symmetric (clean toggle) or irrational hash (organic)
- **Extensible** — add any mesh, set properties, register in handler's SURFACES list

## Scene objects

| Object | Type | Parent | Purpose |
|--------|------|--------|---------|
| `HoloBubbleDome` | MESH (hemisphere) | `HoloBubbleControl` | Dome shell, spherical coords |
| `HoloBubbleFloor` | MESH (disc) | `HoloBubbleControl` | Floor disc, polar coords |
| `HoloBubbleControl` | EMPTY | — | Shared globals, parent for positioning |

Both meshes use blended transparency (`blend_method='BLEND'`,
`surface_render_method='BLENDED'`) with Mix Shader: Transparent BSDF where no
pattern, Emission where pattern lines are active.

## Coordinate systems

Each surface maps its geometry to two abstract axes (X and Y):

- **Dome** (hemisphere): X = phi (pole→equator), Y = theta (around circumference)
- **Floor** (disc): X = radius (center→edge), Y = theta (around circumference)
- **Future planes**: X = horizontal, Y = vertical (cartesian)

Y (theta) is shared across dome and floor so spokes align at the equator/edge.

## Material pipeline (per surface, per axis)

```
Line mask → Wave modulation → Group on/off → Pulse → axis output
```

Two axis outputs combine via `MAXIMUM` (union), then drive emission:

```
max(x_out, y_out) → emission_strength * color → Mix Shader(transparent, emission)
```

### Line mask
`fract(coord * count)` → center on 0.5 → `abs` → `MapRange(0..width → 1..0)`.
Produces a 0/1 mask where the line is.

Also computes line index: `floor(coord * count + 0.5)` — integer ID for group logic.

### Wave modulation (brightness along each line)
`sin(cross_coord * wave_count * 2pi + phase)` mapped from -1..1 to 0.05..1.0.
Multiplied with line mask. Bypassed when `wave_count = 0`.

### Group on/off (which lines are active)

Two modes selectable per axis via `*_pattern_mode`:

**Symmetric (mode 0)**: deterministic cycling
```
on_off = (floor(line_index + step) % unique_patterns) < (unique_patterns / 2)
```
- `unique_patterns=2`: clean alternating toggle (christmas lights)
- `unique_patterns=4`: quarter-rotation stepping
- Step from handler: `floor(t * flips_per_beat)` when snapped

**Irrational hash (mode 1)**: organic pseudo-random
```
on_off = fract(line_index * 0.7236 + step * 0.618) > threshold
```
- Golden ratio spacing ensures maximally different patterns each step
- Threshold (hardcoded 0.3) controls density (~70% on)

### Pulse (overall brightness envelope)
`abs(sin(t * flashes_per_beat * pi)) ** sharpness`
- 0 flashes = constant brightness
- Higher sharpness = more staccato

### Snap control
Per-axis `x_snap` / `y_snap` controls group transition style:
- `snap=1`: step = `floor(raw)` — instant flip (strobe/jerk)
- `snap=0`: step = raw continuous value — smooth sliding transition
- Values 0-1 blend between smooth and snapped

## Control properties

`HoloBubbleControl` is the parent empty for positioning. It has no custom properties —
all control lives on each surface object.

### Per-surface properties (on each mesh object)

All `*_per_beat` and `*_speed` properties support **negative values for reverse**.

| Property | Default | Description |
|----------|---------|-------------|
| **X/Y geometry** | | |
| `x_count` | 12 (dome) / 6 (floor) | Number of X-axis lines |
| `x_width` | 0.03 | X line thickness (0.01=hair, 0.1=thick) |
| `x_offset` | 0 | Slides all X lines along the axis. 0.5=half-cell shift |
| `x_alpha` | 1 | X line opacity: 0=invisible, 0.5=semi-transparent, 1=full |
| `y_count` | 16 | Number of Y-axis lines |
| `y_width` | 0.02 | Y line thickness |
| `y_offset` | 0 | Slides all Y lines along the axis |
| `y_alpha` | 1 | Y line opacity |
| **X/Y color** | | |
| `x_emission` | 8 | X peak emission intensity |
| `x_hue` | 0.55 | X base hue (0=red, 0.33=green, 0.55=cyan) |
| `x_hue_range` | 0 | X hue shift per BarRamp cycle. 0=static |
| `y_emission` | 8 | Y peak emission intensity |
| `y_hue` | 0.55 | Y base hue |
| `y_hue_range` | 0 | Y hue shift per BarRamp cycle |
| **X/Y pulse** | | |
| `x_flashes_per_beat` | 0 | X pulse rate. 0=constant. Negative=reverse |
| `x_flash_sharpness` | 2 | Pulse exponent: 1=smooth, 6=punchy, 8+=strobe |
| **X/Y wave** | | |
| `x_wave_count` | 0 | Sine brightness cycles along each X line. 0=solid |
| `x_wave_speed` | 0 | Wave speed in cycles/beat. 0=frozen. Negative=reverse |
| **X/Y group strobe** | | |
| `x_flips_per_beat` | 4 | Group pattern change rate. 0=frozen. Negative=reverse |
| `x_unique_patterns` | 2 | States before repeating: 2=toggle, 4=quad |
| `x_pattern_mode` | 0 | 0=symmetric (toggle/rotation), 1=hash (organic) |
| `x_snap` | 1 | 0=smooth slide, 1=instant flip. Blend OK |

(Same properties exist for Y axis with `y_` prefix.)

## Frame handler

Registered as `holobubble_handler`. Saved as text block with `use_module = True`.

The handler:
1. Reads `BarRamp.matrix_world.translation.x` (sawtooth 0→1)
2. Maintains a **beat counter** (increments when ramp resets: `r < prev - 0.3`)
3. Computes `t = beat + r_raw` (continuous time in beats)
4. For each registered surface: reads per-object properties, computes all values,
   writes to labeled `ShaderNodeValue` nodes in the material

### Surface registry

```python
SURFACES = [
    ("HoloBubbleDome", "HoloBubbleDomeMat"),
    ("HoloBubbleFloor", "HoloBubbleFloorMat"),
]
```

To add a new surface: create mesh, assign material (via `make_material()`), set
properties (via `set_surface_props()`), add to SURFACES list.

### Value nodes written per material

| Label | Source |
|-------|--------|
| `v_x_count` | `x_count` property |
| `v_x_width` | `x_width` property |
| `v_x_offset` | `x_offset` property |
| `v_x_alpha` | `x_alpha` property |
| `v_x_emission` | `x_emission` property |
| `v_x_hue` | computed: `x_hue + r * x_hue_range` |
| `v_x_wave_count` | `x_wave_count` property |
| `v_x_wave_phase` | computed: `t * wave_speed * 2pi` |
| `v_x_step` | computed: snapped or raw `t * flips` |
| `v_x_unique` | `x_unique_patterns` (min 1) |
| `v_x_hash_step` | same as step |
| `v_x_threshold` | hardcoded 0.3 |
| `v_x_mode` | `x_pattern_mode` |
| `v_x_pulse` | computed: `abs(sin(t*flashes*pi))^sharpness` |

(Same for `v_y_*` prefix.)

## Adding new surfaces

1. Create mesh and parent to `HoloBubbleControl`
2. Call `make_material(name, coord_mode)` — `coord_mode` is `'sphere'`, `'disc'`,
   or extend for `'plane'` (cartesian X/Y)
3. Call `set_surface_props(obj, ...)` to initialize properties with tooltips
4. Add `(obj_name, mat_name)` to handler's `SURFACES` list
5. Re-register handler

The material function handles coordinate mapping; the handler is geometry-agnostic
and processes any surface with the standard property set.

## Duplicating surfaces

Each surface needs its own material (handler writes to value nodes per-material).
To duplicate:
1. Duplicate mesh data and material (`obj.data.copy()`, `mat.copy()`)
2. Create new object, assign duplicated material
3. Copy custom properties from original
4. Add `(new_obj_name, new_mat_name)` to SURFACES and re-register handler

## Known issues and future work

- **World-space coordinates**: Material uses `Geometry > Position` (world space).
  Moving a surface away from world origin shifts the pattern. Future fix: subtract
  object origin in shader, or use Object coordinates with a PatternOrigin empty.
- **Copies require manual registration**: Each duplicate needs its own material and
  a SURFACES entry. Could be automated with a naming convention scan.
- **Hash threshold hardcoded**: Group on/off hash mode uses threshold 0.3 (~70% on).
  Could be exposed as a property if needed.
- **Shape keys**: Dome supports shape keys for morphing (e.g. half-cube, open cylinder,
  flat disc). Create programmatically by repositioning vertices on the basis mesh.
  The emission shader follows deformation since it uses `Geometry > Position`.
- **Cube-face coord_mode**: Current coord_modes (`sphere`, `disc`) use spherical/polar
  coordinates — patterns always look curved even on flat surfaces. Need a `cube` mode
  that picks a planar UV per face based on dominant normal (X/Y/Z face → use the other
  two axes). This enables straight grid lines on cube faces, walls, and floors.
  Subdivided cube + sphere shape key would then give clean cube↔sphere morphing with
  topology-correct patterns on both shapes.
