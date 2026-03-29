# HoloBubble Pattern — Architecture & Reference

Volumetric hologram hemisphere + floor disc enclosing a character. Transparent
emission shader with rings, spokes, wave modulation, and group on/off patterns,
all driven by BarRamp via a frame handler with beat counter.

## Scene objects

| Object | Type | Parent | Purpose |
|--------|------|--------|---------|
| `HoloBubble` | MESH (hemisphere) | `HoloBubble_ctrl` | Dome shell with ring/spoke emission pattern |
| `HoloFloor` | MESH (disc) | `HoloBubble_ctrl` | Floor disc with matching polar pattern |
| `HoloBubble_ctrl` | EMPTY | — | All control properties, parent for positioning |

Both meshes use blended transparency (`blend_method='BLEND'`,
`surface_render_method='BLENDED'`) with Mix Shader: Transparent BSDF where no
pattern, Emission where pattern lines are active.

## Coordinate systems

- **HoloBubble** (hemisphere): spherical coordinates
  - `theta = atan2(y, x) / 2pi` → 0..1 around equator (used by spokes)
  - `phi = acos(z/r) / pi` → 0..1 from pole to equator (used by rings)
- **HoloFloor** (disc): polar coordinates
  - `theta` same as hemisphere (spokes align)
  - `radius = sqrt(x^2 + y^2) / 3.0` → 0..1 center to edge (used by rings)

Spokes share the same `atan2` on both surfaces, so they line up at the equator/edge.

## Material pipeline (per surface)

Each line type (ring, spoke) is an independent pipeline:

```
Line mask → Wave modulation → Group on/off → Pulse → output
```

The two outputs combine via `MAXIMUM` (union), then drive emission:

```
max(ring_output, spoke_output) → emission_strength * color → Mix Shader
```

### Line mask
`fract(coord * count)` → center on 0.5 → `abs` → `MapRange(0..width → 1..0)`.
Produces a 0/1 mask where the line is.

### Wave modulation (brightness along each line)
`sin(cross_coord * wave_count * 2pi + phase)` mapped from -1..1 to 0.05..1.0.
Multiplied with line mask. Makes brightness vary sinusoidally along each line.

### Group on/off (which lines are active)

**TODO: REDESIGN NEEDED** — Current implementation uses `sin(floor(index) * freq)`
which produces smooth gradients, not hard on/off flips. The phase math also suffers
from `sin(n * 2pi) = sin(0)` degeneracy — integer phase steps are invisible.

**Correct approach — symmetric pattern cycling:**

Instead of pseudo-random hashing, define N preset configurations and cycle through
them deterministically. Two new properties per axis:

- `*_group_flips_per_beat` — how many times the pattern switches per beat (strobe rate)
- `*_unique_patterns` — how many distinct states before repeating (2=toggle, 4=quad)

```
step = floor(ramp * flips_per_beat) % unique_patterns   # which pattern state
on_off = floor(line_index + step) % unique_patterns < (unique_patterns / 2)
```

Example with `spoke_unique_patterns=2`, 16 spokes:
- State 0: spokes 0,2,4,6,8,10,12,14 ON — spokes 1,3,5,7,9,11,13,15 OFF
- State 1: the reverse (toggle)

Example with `spoke_unique_patterns=4`, 16 spokes:
- State 0: spokes 0,1,4,5,8,9,12,13 ON
- State 1: shift by 1 — spokes 1,2,5,6,9,10,13,14 ON
- State 2: shift by 2, State 3: shift by 3, then repeats

**Why this is better than irrational hash:**
- Predictable, symmetric, designed-looking (not random)
- Toggle (2) gives classic christmas lights alternation
- Higher values give clean rotational stepping
- Works on any geometry — only needs integer line index
- No sin wrapping, no float precision, no irrational constants

**Shader nodes** (simple integer math):
```
line_index = floor(coord * count + 0.5)        # integer index
shifted = line_index + v_step                   # v_step from handler
mod_val = shifted % unique_patterns             # MODULO node
on_off = mod_val < (unique_patterns / 2)        # LESS_THAN node → hard 0/1
```

**Alternative — irrational hash** (game-of-life / organic patterns):
```
step = floor(ramp * flips_per_beat)
hash = fract(line_index * 0.7236 + step * 0.618)
on_off = hash > threshold
```
Each flip produces a seemingly random but deterministic pattern — like cellular
automata / game of life. Golden ratio (0.618) ensures maximally different patterns
each step. Threshold controls density (0.3 = 70% on, 0.7 = 30% on).

**Switching between modes** via `pattern_mode` property:

| Mode | Value | Effect |
|------|-------|--------|
| Symmetric | 0 | Clean toggle/rotation via modulo cycling |
| Random | 1 | Game-of-life organic flips via irrational hash |

Both use the same `flips_per_beat` for strobe speed. In the shader, a Mix node
or multiply blends between the two on/off outputs based on `v_pattern_mode` (0 or 1).
Separate `ring_pattern_mode` and `spoke_pattern_mode` allow mixing (e.g. symmetric
rings + random spokes).

### Pulse (overall brightness envelope)
`abs(sin(ramp * flashes_per_beat * pi)) ** sharpness`. Independent per ring/spoke.
0 = constant brightness. Always uses raw ramp (not snapped).

## Control properties (on HoloBubble_ctrl)

### Naming convention

All rate/frequency params use `*_per_beat` suffix. Higher = faster. 0 = off/frozen.
Spatial counts use `*_count`. Shape params have descriptive names.

| Property | Default | Description |
|----------|---------|-------------|
| **Ring geometry** | | |
| `ring_count` | 12 | Number of latitude rings (floor gets half) |
| `ring_width` | 0.03 | Ring line thickness (0.01=hair, 0.1=thick) |
| **Ring animation** | | |
| `ring_flashes_per_beat` | 8 | Ring pulse frequency. 0=constant brightness |
| `ring_flash_sharpness` | 2 | Pulse shape: 1=smooth, higher=staccato |
| `ring_wave_count` | 4 | Sine brightness cycles around each ring. 0=solid |
| `ring_wave_steps_per_beat` | 8 | Wave animation rate. 0=frozen |
| `ring_group_count` | 2 | Number of on/off groups across rings. 0=all on |
| `ring_group_flips_per_beat` | 4 | Group pattern reshuffle rate. 0=frozen |
| **Spoke geometry** | | |
| `spoke_count` | 16 | Number of longitude spokes |
| `spoke_width` | 0.02 | Spoke line thickness |
| **Spoke animation** | | |
| `spoke_flashes_per_beat` | 4 | Spoke pulse frequency. 0=constant |
| `spoke_flash_sharpness` | 2 | Pulse shape exponent |
| `spoke_wave_count` | 3 | Sine brightness cycles along each spoke. 0=solid |
| `spoke_wave_steps_per_beat` | 8 | Wave animation rate. 0=frozen |
| `spoke_group_count` | 2 | Number of on/off groups. 0=all on |
| `spoke_group_flips_per_beat` | 4 | Group reshuffle rate (apparent rotation). Negative=reverse |
| **Global** | | |
| `emission_intensity` | 8 | Peak emission brightness |
| `emission_hue_base` | 0.55 | Base hue (0=red, 0.33=green, 0.55=cyan) |
| `emission_hue_range` | 0.15 | Hue shift per BarRamp cycle |
| `transition_snap` | 0 | 0=smooth, 1=snapped to beat. Affects wave/group phases only, not pulse |

## Frame handler

The handler:
1. Reads `BarRamp.matrix_world.translation.x` (sawtooth 0→1)
2. Maintains a **beat counter** (increments when ramp resets)
3. Computes all phases, pulses, and pattern values
4. Writes to labeled `ShaderNodeValue` nodes in both materials

Beat counter detects ramp reset when `r_raw < prev_r - 0.3`.

### Phase calculation

For smooth mode (`transition_snap=0`): continuous `(beat + r_raw) * rate * 2pi`.
For snap mode (`transition_snap=1`): quantized `floor((beat + r_raw) * rate) * (2pi / rate)`.

**TODO**: Replace sin-based group phases with irrational hash approach. The phase
calculation for groups should be:
```python
step = floor((beat + r_raw) * flips_per_beat)  # integer step
# Send 'step' directly to shader, not as a phase angle
# Shader does: fract(line_index * 0.7236 + step * 0.618) > threshold
```

## TODOs for next session

### Critical: Fix strobe/group pattern
1. Replace `sin(index * freq + phase)` group nodes with **symmetric pattern cycling**:
   - Shader: `floor(line_index + v_step) % unique_patterns < unique_patterns / 2` → hard 0/1
   - Handler: send `step = floor((beat + r_raw) * flips_per_beat) % unique_patterns`
   - New properties: `ring_unique_patterns` (default 2), `spoke_unique_patterns` (default 2)
   - `flips_per_beat` = strobe speed, `unique_patterns` = how many states (2=toggle)
   - Works identically for any geometry (hemisphere, floor, cylinder, sphere)

2. Properties already renamed to `*_per_beat` convention (done)

3. Test: `spoke_unique_patterns=2, spoke_group_flips_per_beat=4` should give
   clean alternating spoke toggle 4 times per beat

4. Add `ring_pattern_mode` / `spoke_pattern_mode` (0=symmetric, 1=random/game-of-life)
   - Symmetric: modulo cycling (toggle, quad rotation)
   - Random: irrational hash (organic, cellular automata feel)
   - Both controlled by same `flips_per_beat` for strobe speed

### Geometry-independent strobe design
The strobe system should work on ANY surface by operating on the line index:
- **Input**: integer index of which ring/spoke we're on (`floor(coord * count + 0.5)`)
- **Control**: `flips_per_beat` (rate), `group_count` (how many on/off groups)
- **Output**: 0 or 1 (hard on/off per line)
- **Technique**: irrational hash + golden ratio step (proven in hex beat-driven pattern)
- No sin wrapping, no phase accumulation, no float precision issues

### Future enhancements
- Shape key deformations on HoloBubble (breathing, bulging with beat)
- Geometry nodes modifier for cylinder/sphere morphing
- Additional concentric layers at different radii (text, secondary patterns)
- Volumetric fog glow around the emission lines
- Per-line color variation (not just global hue)
