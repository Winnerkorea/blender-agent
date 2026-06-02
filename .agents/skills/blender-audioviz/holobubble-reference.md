# BubbleDome Pattern — Architecture & Reference

Split-mesh transparent emission system for hemisphere domes. Rings and spokes live on
separate meshes with a tiny normal offset, giving independent pulse/color/animation per
axis with natural depth-based overlap.

## Design principles

- **Split meshes** — one mesh per axis (RingDome, SpokeDome), not one combined mesh.
  Each gets its own material, properties, and can be independently scaled/animated.
- **Normal offset** — RingDome scaled 1.003x so rings float above spokes. Scale is
  keyframeable for expansion/contraction effects on beats or drops.
- **Binary pattern system** — integer values whose bits control which lines/cells are
  on or off. Same bit-extraction math for both gate (which lines) and span (cells along
  lines). Pattern integer tiles naturally if fewer bits than count.
- **Polar naming** — `ring_*` (latitude) and `spoke_*` (longitude/meridians) for
  hemisphere geometry. No abstract x/y naming.
- **Shape keys** — Basis (flat disc), Dome (hemisphere), Cylinder (open tube, no caps).
  Both meshes share the same shape keys. Can be animated independently.

## Scene objects

| Object | Type | Purpose |
|--------|------|---------|
| `SpokeDome` | MESH (hemisphere) | Spoke (longitude) lines only |
| `RingDome` | MESH (hemisphere, 1.003x scale) | Ring (latitude) lines only |
| `AudioGuide` | EMPTY | Kick drum curve for bar marker placement |
| `BarRamp` | EMPTY | Sawtooth 0→1 per bar on X location |
| `Floor` | MESH (circle) | Reflective floor (metallic, low roughness) |

Both dome meshes use blended transparency (`blend_method='BLEND'`,
`surface_render_method='BLENDED'`) with Mix Shader: Transparent BSDF where no
pattern, Emission where pattern lines are active.

## Shape keys

All on both SpokeDome and RingDome (shared topology: 64 segments x 32 rings = 2048 quads):

| Shape key | Description |
|-----------|-------------|
| **Basis** | Flat disc (Z=0, radial spread from R→0) |
| **Dome** | Hemisphere (top ring collapses to pole, degenerate quads = invisible cap) |
| **Cylinder** | Open tube, no top/bottom, full radius walls, evenly spaced Z |

Dome value=1.0 is the primary working state. Animate shape keys for morphing effects.
RingDome scale is independently keyframeable for breathing/expansion effects.

## Coordinate system

Spherical coords computed in shader from world-space Position:

```
ring_coord  = 1 - atan2(z, sqrt(x²+y²)) / (π/2)   → 0 at pole, 1 at equator
spoke_coord = atan2(y, x) / (2π) + 0.5              → 0..1 around circumference
```

Ring span modulates along spoke_coord (cross-axis). Spoke span modulates along ring_coord.

## Properties

Each mesh object has 16 properties for its axis only. No shared globals.

### SpokeDome properties (spoke_*)

| Property | Default | Description |
|----------|---------|-------------|
| **Geometry** | | |
| `spoke_count` | 16 | Number of spoke lines |
| `spoke_width` | 0.02 | Line thickness (0.01=hair, 0.1=thick) |
| `spoke_offset` | 0.0 | Slide along axis (0.5=half-cell shift) |
| `spoke_alpha` | 1.0 | Opacity (0=hidden, 1=full) |
| **Color** | | |
| `spoke_emission` | 8.0 | Peak emission intensity |
| `spoke_hue` | 0.55 | Base hue (0=red, 0.33=green, 0.55=cyan) |
| **Pulse** | | |
| `spoke_pulse_rate` | 0.0 | Flash rate per beat. 0=constant. Negative=reverse |
| `spoke_pulse_sharp` | 2.0 | Exponent (1=smooth, 6=punchy, 8+=strobe) |
| `spoke_pulse_phase` | 0.0 | Phase offset in cycles (0.5=opposite). Rate-independent |
| **Span** (cells along each line) | | |
| `spoke_span_count` | 0 | Cells per line. 0=solid (bypass) |
| `spoke_span_waveform` | 0 | 0=sine, 1=square, 2=saw, 3=saw_reverse |
| `spoke_span_pattern` | 43690 | Binary cell mask (bits=on/off). 43690=alternating 16-bit |
| `spoke_span_rate` | 0.0 | Cell scroll rate per beat. Negative=reverse |
| **Gate** (which lines are on/off) | | |
| `spoke_gate_rate` | 4.0 | Pattern rotation rate per beat. Negative=reverse |
| `spoke_gate_pattern` | 43690 | Binary line mask (bits=on/off, rotates over time) |
| `spoke_gate_snap` | 1.0 | 0=smooth slide, 1=instant flip |

### RingDome properties (ring_*)

Same structure with `ring_` prefix. Different defaults:

| Property | Default | Notes |
|----------|---------|-------|
| `ring_count` | 8 | Hemisphere = half sphere, 8 rings is right density |
| `ring_gate_pattern` | 170 | 10101010 — alternating 8-bit |

All other defaults match spoke equivalents.

## Binary pattern system

Both gate and span use the same bit-extraction math. An integer's binary representation
controls which lines/cells are on or off:

### Shader math (same for gate and span)
```
bit = floor(pattern / pow(2, (index + step) % count)) % 2
```

### Pattern tiling
Pattern tiles naturally based on significant bits. `pattern=170` (8-bit 10101010) with
`count=16` tiles twice. Use wider patterns for full control: `43690` = 16-bit alternating.

### Useful pattern values

**For 16 items (spoke_count=16):**

| Value | Binary | Effect |
|-------|--------|--------|
| 65535 | 1111111111111111 | All on |
| 43690 | 1010101010101010 | Alternating |
| 52428 | 1100110011001100 | Pairs |
| 61166 | 1110111011101110 | 3-on 1-off |
| 4369 | 0001000100010001 | Quarter (cross/compass) |
| 257 | 0000000100000001 | Two opposing |

**For 8 items (ring_count=8):**

| Value | Binary | Effect |
|-------|--------|--------|
| 255 | 11111111 | All on |
| 170 | 10101010 | Alternating |
| 204 | 11001100 | Pairs |
| 105 | 01101001 | Irregular |

### Waveform types (span only)

Within each "on" cell, the waveform shapes brightness:

| Value | Name | Shape |
|-------|------|-------|
| 0 | sine | Smooth fade 0→1→0 (half sine) |
| 1 | square | Full brightness across cell |
| 2 | saw | Ramp 0→1 (bright at trailing edge) |
| 3 | saw_reverse | Ramp 1→0 (bright at leading edge) |

Use `saw_reverse` (3) when brightness should lead the rotation direction.

## Pulse phase

Phase is **cycle-relative**, not beat-relative. `0.5` always means opposite pulse
regardless of rate. This avoids the trap where beat-relative phase cancels out at
certain rates (e.g. phase=0.5 with rate=4 adds exactly 2π = no shift).

Formula: `abs(sin((t * rate + phase) * π)) ^ sharpness`

## Material pipeline

Each material (SpokeDomeMat / RingDomeMat) contains only its own axis. No vestigial
nodes from the other axis.

```
Geometry Position → spherical coords (atan2/acos) → ring_coord + spoke_coord
                                                          ↓
                                           line_mask (fract → center → MapRange)
                                                          ↓
                                           × gate_bit (binary pattern extraction)
                                                          ↓
                                           × span (waveform × binary cell pattern)
                                                          ↓
                                           × pulse (from handler)
                                                          ↓
                                           × alpha
                                                          ↓
                                     Mix Shader (Transparent ↔ Emission)
```

Value nodes labeled `v_<prefix>_<param>` are written by the frame handler each frame.

### Span bypass

When `span_count=0`, handler sets `v_*_span_bypass=1.0` and `v_*_span_count=1.0`
(avoids division by zero). The bypass value adds to the span output (which is 0),
giving 1.0 = no modulation. When span is active, bypass=0.0.

## Frame handler

Registered as `bubbledome_handler`. Saved as text block `bubbledome_handler.py` with
`use_module = True` (auto-runs on file load).

### Surface registry

```python
SURFACES = [
    ("SpokeDome", "SpokeDomeMat"),
    ("RingDome", "RingDomeMat"),
]
```

### Beat counter

Handler maintains a beat counter that increments when BarRamp resets (current < previous - 0.3).
Continuous time `t = beat_count + ramp_raw` drives all temporal computations.

### Per-frame computation

For each surface, the handler:
1. Reads all custom properties from the mesh object
2. Computes pulse: `abs(sin((t * rate + phase) * π)) ^ sharpness`
3. Computes gate step: `floor(t * gate_rate)` when snapped, raw when smooth
4. Computes span phase: `t * span_rate`
5. Writes all values to labeled Value nodes in the material

**Do NOT call `depsgraph.update()`** in frame handlers — causes crashes.

## Adding new surfaces

1. Create mesh, optionally parent for positioning
2. Duplicate an existing material and rename
3. Set properties on the new object (only the axis it displays)
4. Add `(obj_name, mat_name)` to handler's `SURFACES` list
5. Re-register handler

Each surface needs its own material instance (handler writes to value nodes per-material).

## Known issues and future work

- **World-space coordinates**: Material uses `Geometry > Position` (world space).
  Moving a surface away from world origin shifts the pattern. Future fix: subtract
  object origin in shader.
- **Hue blending**: Currently handler writes a single hue per material. Per-pixel hue
  blending between axes would require shader-level mixing (not needed with split meshes).
- **Floor surface**: Could add a third mesh (floor disc) with polar coordinate mapping
  (radius/theta) and its own material + properties. Register in SURFACES list.
- **Cube morphing**: Subdivided cube with sphere shape key could give cube↔dome morphing.
  Would need a `cube` coord_mode for straight grid lines on flat faces.
