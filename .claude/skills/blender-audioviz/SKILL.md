---
name: blender-audioviz
description: Audio visualization for Blender — extract frequency curves from audio files, create music-reactive pattern materials (grid, checkerboard, hex, radial, concentric, polar), set up stage lighting rigs driven by audio, and wire drivers between audio curves and any material or light property. Use when the user mentions audio visualization, music-driven animation, beat-synced effects, stage lighting, or reactive materials in Blender.
---

# Blender Audio Visualization Skill

Extract audio frequency curves, create music-reactive pattern materials, and set up
stage lighting rigs. EEVEE primary. Send all code via:
```bash
curl -s localhost:5656 --data-binary @- <<'PYEOF'
<python code>
PYEOF
```
See `blender` skill for communication details. See `blender-3d` for materials, drivers,
cameras, and rendering. See `blender-laser` for laser beam integration.

## Architecture

Three control objects anchor the system:

- **AudioGuide** — Empty with kick drum curve baked as custom property. Reference only —
  user views in Graph Editor alongside audio in Sequencer to place bar markers.
- **BarRamp** — Empty whose X location is a sawtooth 0→1 per bar (keyframed at bar
  markers). All beat-synced effects derive from this single ramp via sine math.
  Self-contained per bar — no drift accumulation if a bar marker is slightly off.
- **PatternOrigin** — Empty used as the texture coordinate object for all pattern materials.
  Move/rotate it to move/rotate all patterns coherently. Per-surface offset via each
  surface mesh's own object origin.

**Pulse light control**: Each light gets a control empty (`LightName_ctrl`) where
transform channels store keyframeable parameters:

| Channel | Parameter | Description |
|---------|-----------|-------------|
| X location | `mult` | BPM multiplier: `N*π` gives `N/2` pulses/bar. 4=quarter, 8=8th, 16=16th |
| Y location | `sharpness` | Sine exponent: 2=soft, 6=punchy, 8-12=hard strobe |
| Z location | `intensity` | Peak emission (watts for lights, strength for materials) |

Keyframe X with **CONSTANT** interpolation at bar boundaries to change rate for drops.
All parameters use TRANSFORMS drivers (not custom properties — those don't evaluate
reliably in Blender drivers).

**Standard property interface** on each surface object wearing a pattern material:

| Property | Type | Default | Controls |
|----------|------|---------|----------|
| `brightness` | float | 1.0 | Emission strength (0-10) |
| `hue` | float | 0.5 | HSV hue (0-1 cyclic, drives Hue Saturation Value node) |
| `speed` | float | 0.0 | Pattern scroll speed (drives Mapping offset over time) |
| `flash` | float | 0.0 | Flash intensity (0-1, multiplied into emission) |

Each pattern type adds its own extras (e.g. `line_thickness`, `beam_count`).

## Audio Extraction

### Step 1: Convert to WAV

Always convert to WAV for reliable scrubbing and analysis. Requires `ffmpeg` on PATH.

```python
import subprocess, os

def convert_to_wav(input_path, output_dir):
    """Convert any audio file to 44.1kHz mono WAV."""
    base = os.path.splitext(os.path.basename(input_path))[0]
    wav_path = os.path.join(output_dir, f"{base}.wav")
    subprocess.run([
        "ffmpeg", "-y", "-i", input_path,
        "-ar", "44100", "-ac", "1", "-sample_fmt", "s16",
        wav_path
    ], check=True, capture_output=True)
    return wav_path
```

Run this **outside Blender** (in the shell), not via the Blender HTTP server — ffmpeg
subprocess inside Blender can hang. Use `OUTPUT` for the output directory:
```bash
# Convert OGG to WAV
ffmpeg -y -i /path/to/song.ogg -ar 44100 -ac 1 -sample_fmt s16 output/song.wav
```

### Step 2: Extract kick drum curve

Extract only the kick band (40-100Hz) — this is the primary reference for placing
bar markers. Other bands can be added later if needed, but kick alone is sufficient
for beat detection. Run **inside Blender** (via HTTP). Uses `wave` + `numpy` (bundled).

```python
import bpy, wave, numpy as np

def extract_kick(wav_path, fps, frame_count):
    """Extract kick drum (40-100Hz) amplitude per frame using FFT."""
    wf = wave.open(wav_path, 'rb')
    sr = wf.getframerate()
    raw = wf.readframes(wf.getnframes())
    wf.close()
    audio = np.frombuffer(raw, dtype=np.int16).astype(np.float64) / 32768.0
    spf = sr // fps
    window_size = 2048

    kick_lo, kick_hi = 40, 100
    values = []
    for i in range(frame_count):
        start = i * spf
        chunk = audio[start:start + window_size]
        if len(chunk) < window_size:
            values.append(0.0)
            continue
        fft = np.fft.rfft(chunk)
        freqs = np.fft.rfftfreq(window_size, 1.0 / sr)
        magnitudes = np.abs(fft)
        mask = (freqs >= kick_lo) & (freqs <= kick_hi)
        values.append(float(np.mean(magnitudes[mask])) if mask.any() else 0.0)

    # Normalize to 0-1, then sharpen transients (power 2.0)
    mx = max(values) if values else 1.0
    if mx > 0:
        values = [v / mx for v in values]
    values = [v ** 2.0 for v in values]
    mx2 = max(values) if values else 1.0
    if mx2 > 0:
        values = [v / mx2 for v in values]

    return values
```

### Step 3: Bake to AudioGuide object

AudioGuide is a reference-only empty — the user views the kick curve in the Graph
Editor to identify bar boundaries and place markers.

```python
import bpy

def bake_kick_to_guide(values, frame_start=1):
    """Bake kick curve as keyframes on AudioGuide empty."""
    guide = bpy.data.objects.get("AudioGuide")
    if not guide:
        guide = bpy.data.objects.new("AudioGuide", None)
        bpy.context.scene.collection.objects.link(guide)
        guide.empty_display_type = 'ARROWS'
        guide.empty_display_size = 0.3

    for i, val in enumerate(values):
        guide["kick"] = val
        guide.keyframe_insert(data_path='["kick"]', frame=frame_start + i)

    if guide.animation_data and guide.animation_data.action:
        for fc in guide.animation_data.action.fcurves:
            for kp in fc.keyframe_points:
                kp.interpolation = 'LINEAR'

    return guide
```

Optionally create a binary gate curve for clearer visual reference:

```python
def bake_kick_gate(guide, threshold=0.25, frame_start=1):
    """Binary 0/1 curve: 1 when kick >= threshold, else 0. CONSTANT interp."""
    action = guide.animation_data.action
    kick_fc = next(fc for fc in action.fcurves if fc.data_path == '["kick"]')
    kick_values = [(kp.co[0], kp.co[1]) for kp in kick_fc.keyframe_points]

    guide["kick_gate"] = 0.0
    for frame, val in kick_values:
        guide["kick_gate"] = 1.0 if val >= threshold else 0.0
        guide.keyframe_insert(data_path='["kick_gate"]', frame=frame)

    gate_fc = next(fc for fc in action.fcurves if fc.data_path == '["kick_gate"]')
    for kp in gate_fc.keyframe_points:
        kp.interpolation = 'CONSTANT'
```

### Step 4: Load audio into sequencer

```python
scene = bpy.context.scene
if not scene.sequence_editor:
    scene.sequence_editor_create()
se = scene.sequence_editor
sound = se.strips.new_sound(name="Music", filepath=wav_path, channel=1, frame_start=1)
sound.volume = 1.0
```

User can now play back the timeline and see AudioGuide curves in the Graph Editor
alongside the audio waveform in the Sequencer. The curves are **reference only** — the
user hand-keyframes the actual control properties using the guide curves as visual reference.

### Verifying audio curves

**Ask the user to play back the timeline** rather than rendering video or taking many
screenshots. The user can see the curves in the Graph Editor, hear the audio, and
watch the viewport in real-time — this is far more informative than any render for
validating sync and feel. Ask them:
- "Play the timeline and check if the kick curve matches the beat"
- "Does the floor pulse feel in sync with the music?"
- "Are the snare hits distinct enough or too muddy?"

Only render a frame or short clip when the user confirms the curves feel right and
wants to check the visual output.

### Complete setup workflow

```bash
# 1. Outside Blender (shell) — convert audio to WAV:
ffmpeg -y -i /path/to/song.ogg -ar 44100 -ac 1 -sample_fmt s16 /project/song.wav
```

```python
# 2. Inside Blender (via HTTP) — set up scene, extract kick, load audio:
import bpy
scene = bpy.context.scene
scene.render.fps = 60
scene.frame_end = int(duration_seconds * 60)  # calculate from audio duration

wav_path = "/project/song.wav"
values = extract_kick(wav_path, scene.render.fps,
                      scene.frame_end - scene.frame_start + 1)
guide = bake_kick_to_guide(values, frame_start=scene.frame_start)

# Load audio into sequencer for playback sync
if not scene.sequence_editor:
    scene.sequence_editor_create()
scene.sequence_editor.strips.new_sound(
    name="Music", filepath=wav_path, channel=1, frame_start=scene.frame_start)
```

```
3. User places bar markers (b00, b01, ...) using kick curve as guide.
   LLM analyzes spacing, interpolates missing bars, extrapolates 1-2 beyond.

4. LLM creates BarRamp sawtooth from markers:
   bar_frames = [m.frame for m in sorted(scene.timeline_markers, key=lambda m: m.frame)]
   ramp = create_bar_ramp(bar_frames)

5. LLM creates pulse lights with control empties.
   User keyframes mult/sharpness/intensity on control empties for drops.
```

### Bar markers and BPM analysis

After baking kick curves, the user places timeline markers at bar boundaries. The LLM
can analyze marker spacing to estimate BPM and verify even spacing:

```python
markers = sorted(bpy.context.scene.timeline_markers, key=lambda m: m.frame)
fps = bpy.context.scene.render.fps
for i in range(1, len(markers)):
    df = markers[i].frame - markers[i-1].frame
    bpm = 60.0 / (df / fps) * 4  # assuming 4 beats per bar
    print(f"{markers[i].name}: {df} frames, ~{bpm:.1f} BPM")
```

If the user skips a bar (e.g. no kick drum in that section), interpolate by averaging
the adjacent bars. Also extrapolate 1-2 bars before the first and after the last marker.

### BarRamp — sawtooth ramp

A per-bar sawtooth ramp (0→1) on the **BarRamp** empty's X location. All beat-synced
effects derive from this single ramp. Self-contained per bar — no drift accumulation.

**Why X location instead of a custom property**: Custom properties do not evaluate
reliably in Blender drivers (the value gets stuck at the last-set value). Object
transforms always interpolate correctly. Use TRANSFORMS driver variables to read them.

```python
import bpy

def create_bar_ramp(bar_frames):
    """Create BarRamp empty with sawtooth 0→1 per bar on X location."""
    scene = bpy.context.scene
    ramp = bpy.data.objects.get("BarRamp")
    if not ramp:
        ramp = bpy.data.objects.new("BarRamp", None)
        scene.collection.objects.link(ramp)
        ramp.empty_display_type = 'SINGLE_ARROW'
        ramp.empty_display_size = 0.3

    if ramp.animation_data:
        ramp.animation_data_clear()

    for i in range(len(bar_frames)):
        start = bar_frames[i]
        end = bar_frames[i + 1] if i + 1 < len(bar_frames) \
              else start + (bar_frames[-1] - bar_frames[-2])

        # One frame before bar: ramp reaches 1.0 (end of previous ramp)
        if start > scene.frame_start:
            ramp.location.x = 1.0
            ramp.keyframe_insert(data_path="location", index=0, frame=start - 1)

        # Bar start: instant drop to 0
        ramp.location.x = 0.0
        ramp.keyframe_insert(data_path="location", index=0, frame=start)

    # End of last bar
    last_len = bar_frames[-1] - bar_frames[-2]
    end_frame = bar_frames[-1] + last_len
    ramp.location.x = 1.0
    ramp.keyframe_insert(data_path="location", index=0, frame=min(end_frame, scene.frame_end))

    # LINEAR interpolation on all keyframes
    fc = ramp.animation_data.action.fcurves[0]
    for kp in fc.keyframe_points:
        kp.interpolation = 'LINEAR'

    return ramp
```

### Pulse math

The core pulse formula used in all drivers:

```
max(0, abs(sin(ramp * mult * π)) ** sharpness * intensity - threshold)
```

- `ramp` = BarRamp X location (0→1 per bar)
- `mult` = N gives N/2 pulses per bar (4=quarter, 8=8th, 16=16th, 32=32nd)
- `sharpness` = exponent (2=soft sine, 6=punchy, 8+=hard strobe)
- `intensity` = peak output value
- `threshold` = small value (0.3) to cut sine tail to hard zero

The sine evaluates at full precision — not quantized to frames. This means
32nd notes at 60fps still produce clean spikes.

## Control Object Setup

```python
import bpy

# AudioGuide — reference curves only (kick drum for bar marker placement)
guide = bpy.data.objects.new("AudioGuide", None)
bpy.context.scene.collection.objects.link(guide)
guide.empty_display_type = 'ARROWS'
guide.empty_display_size = 0.3

# BarRamp — sawtooth 0→1 per bar (created after user places bar markers)
# See create_bar_ramp() in "BarRamp — sawtooth ramp" section

# PatternOrigin — shared texture coordinate source for pattern materials
origin = bpy.data.objects.new("PatternOrigin", None)
bpy.context.scene.collection.objects.link(origin)
origin.empty_display_type = 'PLAIN_AXES'
origin.empty_display_size = 1.0
```

### Adding standard properties to a surface object

```python
def add_pattern_controls(obj, extras=None):
    """Add standard audio-viz control properties to an object."""
    obj["brightness"] = 1.0
    obj["hue"] = 0.5
    obj["speed"] = 0.0
    obj["flash"] = 0.0
    if extras:
        for name, default in extras.items():
            obj[name] = default
```

## Pattern Materials

Six pattern types available. All share:
- **Object coordinates** from PatternOrigin empty
- **HSV color pipeline** (hue 0-1 cyclic)
- **Emission output** for glow in dark scenes
- **Standard property interface** (brightness, hue, speed, flash)

| Pattern | Technique | Specific extras |
|---------|-----------|-----------------|
| **Grid** | Brick Texture | `line_thickness`, `tile_size`, `color_mix` |
| **Checkerboard** | Modulo + threshold | `cell_size`, `contrast` |
| **Hexagonal** | Hex offset rows + distance | `cell_size`, `border_width` |
| **Hex Beat-Driven** | Hex + irrational cell hash + BarRamp | Selective cell lighting, symmetric, snaps on beat |
| **Radial** | ATAN2 + sine + threshold | `beam_count`, `beam_width` |
| **Concentric** | Distance + sine + threshold | `ring_count`, `ring_width` |
| **Polar** | Normal vector + radial math | `rings`, `spokes` |
| **Circle Polar** | Rings + spokes + frame handler | Independent ring/spoke flash, pattern, grid controls |
| **HoloBubble** | Geometry-agnostic axis system, emission + transparent | Per-surface x/y props, wave, group strobe, snap, reverse |

See `pattern-reference.md` for complete Python code for each pattern.

The **HoloBubble** pattern is a geometry-agnostic transparent emission system. Each
surface object owns its own `x_*`/`y_*` properties (count, width, pulse, wave, group
strobe, snap, pattern mode). A single frame handler processes all registered surfaces.
Supports negative rates for reverse direction and per-axis snap (0=smooth, 1=instant).
See `holobubble-reference.md` for architecture, property reference, and how to add
new surfaces.

### Shared material preamble

All pattern materials start with this structure:

```python
import bpy

def create_pattern_base(name, obj, origin_empty, use_normal=False):
    """Create base material with Object coords, HSV pipeline, emission output.
    Returns (material, nodes, links, sep_xyz_node) for pattern-specific wiring."""
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    nodes = mat.node_tree.nodes
    links = mat.node_tree.links
    for n in list(nodes):
        nodes.remove(n)

    # Output
    output = nodes.new("ShaderNodeOutputMaterial")
    output.location = (800, 0)

    # Principled BSDF for emission
    bsdf = nodes.new("ShaderNodeBsdfPrincipled")
    bsdf.label = "pattern_surface"
    bsdf.location = (500, 0)
    bsdf.inputs["Base Color"].default_value = (0.02, 0.02, 0.02, 1)
    bsdf.inputs["Metallic"].default_value = 0.5
    bsdf.inputs["Roughness"].default_value = 0.2
    links.new(output.inputs["Surface"], bsdf.outputs["BSDF"])

    # HSV node for hue control
    hsv = nodes.new("ShaderNodeHueSaturation")
    hsv.label = "color_control"
    hsv.location = (300, 100)
    hsv.inputs["Saturation"].default_value = 1.0
    hsv.inputs["Value"].default_value = 1.0
    links.new(bsdf.inputs["Emission Color"], hsv.outputs["Color"])

    # Base emission color (will be modulated by HSV hue)
    hsv.inputs["Color"].default_value = (0.0, 0.5, 1.0, 1)  # cyan base

    # Brightness → Emission Strength (driven by property)
    bright_val = nodes.new("ShaderNodeValue")
    bright_val.label = "brightness"
    bright_val.location = (300, -100)
    bright_val.outputs[0].default_value = 1.0
    links.new(bsdf.inputs["Emission Strength"], bright_val.outputs[0])

    # Texture coordinates
    tex_coord = nodes.new("ShaderNodeTexCoord")
    tex_coord.location = (-600, 0)
    if not use_normal:
        tex_coord.object = origin_empty

    # Mapping node (speed drives Location offset)
    mapping = nodes.new("ShaderNodeMapping")
    mapping.label = "pattern_mapping"
    mapping.location = (-400, 0)
    coord_output = "Normal" if use_normal else "Object"
    links.new(mapping.inputs["Vector"], tex_coord.outputs[coord_output])

    # Separate XYZ for pattern math
    sep = nodes.new("ShaderNodeSeparateXYZ")
    sep.location = (-200, 0)
    links.new(sep.inputs["Vector"], mapping.outputs["Vector"])

    # Assign material
    obj.data.materials.clear()
    obj.data.materials.append(mat)

    return mat, nodes, links, sep, hsv, bright_val, mapping
```

### Connecting the pattern mask

After building pattern-specific nodes that produce a 0-1 mask:

```python
# Mix: mask selects between black (no emission) and the HSV-controlled color
mix = nodes.new("ShaderNodeMix")
mix.data_type = 'RGBA'
mix.location = (150, 100)
mix.inputs["A"].default_value = (0, 0, 0, 1)      # off
mix.inputs["B"].default_value = (0, 0.5, 1.0, 1)  # base color (same as HSV input)
links.new(mix.inputs["Factor"], pattern_mask.outputs[0])
links.new(hsv.inputs["Color"], mix.outputs["Result"])
```

## Pulse Light System

### Creating a pulse light

Each light gets a control empty whose transform channels store keyframeable parameters.
All use TRANSFORMS driver variables (never custom properties — they don't evaluate
reliably in Blender drivers).

```python
import bpy

def create_pulse_light(name, location, aim_target, ramp_obj,
                       color=(1,1,1), mult=4, sharpness=6, intensity=800):
    """Create a spot light + control empty driven by BarRamp.

    Control empty channels (keyframeable):
      X = mult (BPM multiplier: 4=quarter, 8=8th, 16=16th, 32=32nd)
      Y = sharpness (exponent: 2=soft, 6=punchy, 8+=hard strobe)
      Z = intensity (peak emission watts)
    """
    # Spot light
    bpy.ops.object.light_add(type='SPOT', location=location)
    spot = bpy.context.active_object
    spot.name = name
    spot.data.spot_size = 0.8
    spot.data.spot_blend = 0.3
    spot.data.color = color
    spot.data.shadow_soft_size = 0.2

    track = spot.constraints.new('TRACK_TO')
    track.target = aim_target
    track.track_axis = 'TRACK_NEGATIVE_Z'
    track.up_axis = 'UP_Y'

    # Control empty — transform channels ARE the parameters
    ctrl = bpy.data.objects.new(f"{name}_ctrl", None)
    bpy.context.scene.collection.objects.link(ctrl)
    ctrl.empty_display_type = 'PLAIN_AXES'
    ctrl.empty_display_size = 0.3
    ctrl.location = (mult, sharpness, intensity)
    ctrl.hide_render = True

    # Driver: energy = pulse(ramp, mult, sharpness, intensity)
    drv = spot.data.driver_add("energy")

    var_r = drv.driver.variables.new()
    var_r.name = "r"
    var_r.type = 'TRANSFORMS'
    var_r.targets[0].id = ramp_obj
    var_r.targets[0].transform_type = 'LOC_X'
    var_r.targets[0].transform_space = 'WORLD_SPACE'

    var_m = drv.driver.variables.new()
    var_m.name = "m"
    var_m.type = 'TRANSFORMS'
    var_m.targets[0].id = ctrl
    var_m.targets[0].transform_type = 'LOC_X'
    var_m.targets[0].transform_space = 'WORLD_SPACE'

    var_s = drv.driver.variables.new()
    var_s.name = "s"
    var_s.type = 'TRANSFORMS'
    var_s.targets[0].id = ctrl
    var_s.targets[0].transform_type = 'LOC_Y'
    var_s.targets[0].transform_space = 'WORLD_SPACE'

    var_i = drv.driver.variables.new()
    var_i.name = "i"
    var_i.type = 'TRANSFORMS'
    var_i.targets[0].id = ctrl
    var_i.targets[0].transform_type = 'LOC_Z'
    var_i.targets[0].transform_space = 'WORLD_SPACE'

    drv.driver.expression = "max(0, abs(sin(r * m * 3.14159265)) ** s * i - 0.3)"

    return spot, ctrl
```

### Same pattern for emissive meshes

For cubes, planes, or any mesh with emission material, drive emission strength
instead of light energy. The driver expression is identical:

```python
mat = obj.data.materials[0]
bsdf = mat.node_tree.nodes["Principled BSDF"]
# Clear existing material drivers
if mat.node_tree.animation_data:
    mat.node_tree.animation_data_clear()

drv = bsdf.inputs["Emission Strength"].driver_add("default_value")
# ... add r, m, s, i variables same as above ...
drv.driver.expression = "max(0, abs(sin(r * m * 3.14159265)) ** s * i - 0.3)"
```

### Keyframing drops

Change BPM multiplier at bar boundaries with CONSTANT interpolation for instant
rate switches. Example: verse at mult=8, drop at mult=32:

```python
ctrl = bpy.data.objects["PulseSpot_1_ctrl"]
markers = {m.name: m.frame for m in bpy.context.scene.timeline_markers}

# Verse: 8th notes
ctrl.location.x = 8
ctrl.keyframe_insert(data_path="location", index=0, frame=markers["b00"])

# Drop: 32nd notes + brighter
ctrl.location.x = 32
ctrl.keyframe_insert(data_path="location", index=0, frame=markers["b04"])
ctrl.location.z = 1500  # intensity bump
ctrl.keyframe_insert(data_path="location", index=2, frame=markers["b04"])

# Back to verse
ctrl.location.x = 8
ctrl.keyframe_insert(data_path="location", index=0, frame=markers["b06"])
ctrl.location.z = 600
ctrl.keyframe_insert(data_path="location", index=2, frame=markers["b06"])

# CONSTANT interpolation — snap, don't blend
for fc in ctrl.animation_data.action.fcurves:
    for kp in fc.keyframe_points:
        kp.interpolation = 'CONSTANT'
```

### Mult reference

| mult value | Pulses per bar | Musical term |
|------------|---------------|--------------|
| 4 | 2 | half notes |
| 8 | 4 | quarter notes |
| 16 | 8 | 8th notes |
| 32 | 16 | 16th notes |
| 64 | 32 | 32nd notes |

### Wiring to character materials

To make any existing material pulse-reactive, drive its emission strength from BarRamp:

```python
mat = bpy.data.materials["clothes01"]
bsdf = mat.node_tree.nodes["Principled BSDF"]
ramp = bpy.data.objects["BarRamp"]

drv = bsdf.inputs["Emission Strength"].driver_add("default_value")
var = drv.driver.variables.new()
var.name = "r"
var.type = 'TRANSFORMS'
var.targets[0].id = ramp
var.targets[0].transform_type = 'LOC_X'
var.targets[0].transform_space = 'WORLD_SPACE'
drv.driver.expression = "max(0, abs(sin(r * 8 * 3.14159265)) ** 6 * 3 - 0.1)"
```

For bulk application to all materials on a character armature, loop over child
meshes and apply the same driver pattern.

### Shadow suppression (optional)

Emissive floor patterns can cast unwanted shadows. Suppress with Light Path trick:

```python
# Add to any pattern material
light_path = nodes.new("ShaderNodeLightPath")
transparent = nodes.new("ShaderNodeBsdfTransparent")
mix_shadow = nodes.new("ShaderNodeMixShader")
mix_shadow.label = "Shadow_Suppress"

links.new(mix_shadow.inputs["Fac"], light_path.outputs["Is Shadow Ray"])
links.new(mix_shadow.inputs[1], bsdf.outputs["BSDF"])        # normal surface
links.new(mix_shadow.inputs[2], transparent.outputs["BSDF"])  # transparent for shadows
links.new(output.inputs["Surface"], mix_shadow.outputs["Shader"])
```

## Frame Handler Pattern (preferred for complex surfaces)

For surfaces with many parameters (like the polar floor), use a **frame handler**
instead of drivers. The handler reads custom properties via Python (which works
correctly) and writes to labeled Value nodes in the shader.

### Why frame handlers over drivers

- **Custom properties work**: Python reads them fine; only Blender's driver evaluator
  has the bug where values get stuck.
- **Readable names**: `pulse_ring_flash` instead of overloading location/rotation/scale.
- **Centralized logic**: All math in one Python function, easy to modify globally.
- **No channel limits**: Drivers are limited to 9 transform channels per control empty.

### Persistent handler script

Save the handler as a **text block** in the .blend file with `use_module = True` so
it auto-runs on file load. Blender will prompt "Run Python scripts" — user must accept.

```python
# Create text block
txt = bpy.data.texts.get("my_handler.py")
if txt:
    bpy.data.texts.remove(txt)
txt = bpy.data.texts.new("my_handler.py")
txt.write(handler_source_code)
txt.use_module = True  # auto-run on file load
```

### Handler template

```python
import bpy
from math import sin, pi, floor as mfloor, fabs

handler_name = "my_handler"

def my_update(scene, depsgraph=None):
    ctrl = bpy.data.objects.get("MyCtrl")
    ramp = bpy.data.objects.get("BarRamp")
    mat = bpy.data.materials.get("MyMat")
    if not ctrl or not ramp or not mat:
        return

    nodes = mat.node_tree.nodes
    r = ramp.matrix_world.translation.x  # BarRamp X (always correct)

    # Read custom properties (Python reads these fine)
    flash = ctrl.get("pulse_flash", 8.0)
    sharpness = ctrl.get("pulse_sharpness", 6.0)

    # Compute
    flash_val = fabs(sin(r * flash * pi)) ** sharpness

    # Write to labeled Value nodes
    for nd in nodes:
        if nd.label == "my_value":
            nd.outputs[0].default_value = flash_val

my_update._name = handler_name

def register():
    for h in list(bpy.app.handlers.frame_change_post):
        if hasattr(h, "_name") and h._name == handler_name:
            bpy.app.handlers.frame_change_post.remove(h)
    bpy.app.handlers.frame_change_post.append(my_update)

register()
```

**Do NOT call `depsgraph.update()`** in frame handlers — causes crashes (GIL
contention). Just set values directly; Blender picks them up on the next draw.

### When to use drivers vs frame handlers

| Use case | Approach |
|----------|----------|
| Simple lights (1-3 params) | TRANSFORMS drivers on control empties |
| Complex surfaces (5+ params) | Frame handler with custom properties |
| Character material wiring | TRANSFORMS driver (one expression per material) |
| Anything needing readable names | Frame handler |

## Known Driver Pitfalls

- **Custom properties don't evaluate in drivers**: Values get stuck at the last
  Python-set value. Always use TRANSFORMS variables (object location/rotation/scale)
  for driver inputs. Store parameters as transform channels on control empties.
- **SINGLE_PROP driver variables**: Same issue as custom properties. Avoid for
  anything that needs to interpolate between keyframes.
- **Workaround**: Use frame handlers instead — Python reads custom properties
  correctly and writes to Value nodes in the shader.

## Lighting Rigs

Three presets with a shared interface. See `lighting-rigs.md` for complete setup code.
Each rig's lights can be wired to the pulse system via `create_pulse_light()` or by
adding the TRANSFORMS driver pattern to existing lights.

| Preset | Description | Best for |
|--------|-------------|----------|
| **Ring** | N spots on a circle, all aiming at a shared target | Concert stage beams, sweeping searchlights |
| **Cluster** | N spots at one point with angular offsets | Concentrated beam bundle, fan-out effects |
| **Disco Ball** | Faceted icosphere + orbiting small spots | Scattered light dots sweeping across room |

### Common parameters

| Param | Type | Description |
|-------|------|-------------|
| `count` | int | Number of lights (4-16 typical) |
| `radius` | float | Distance from center (ring/cluster spacing) |
| `focus` | float | Beam convergence: positive=converge at center, negative=fan outward |
| `energy` | float | Light intensity (watts) |
| `hue` | float | HSV hue (0-1 cyclic) for color |
| `saturation` | float | Color saturation (0-1) |
| `sweep_angle` | float | Rotation sweep for animation (radians) |

**Fog volume** is a separate setup step — see `blender-3d` skill volumetric section.
Light rigs do NOT auto-create fog.

## Laser Integration

For laser beams driven by audio, see the `blender-laser` skill. Key wiring points:

| Audio curve | Laser parameter | Effect |
|-------------|----------------|--------|
| `hihat` or `snare` | Pivot rotation speed | Fast beam sweeping on percussion hits |
| `master_energy` | Beam color hue | Slow color cycling with overall energy |
| `kick` | Beam emission strength | Brightness pulse on kick drum |
| Beat drop detection | Bounce count | More reflections during intense sections |

Wire via drivers on the LaserPivot object and beam materials — see `blender-laser`
skill for the handler setup pattern.

## Creative Ideas

See `ideas.md` for advanced techniques: floor displacement, particle bursts,
ribbon trails, beat-synced camera cuts, silhouette mode, reflective floors,
stage transitions, and more. Each idea notes complexity and which skills it builds on.
