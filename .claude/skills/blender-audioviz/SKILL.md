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

Two control objects anchor the system:

- **AudioGuide** — Empty with audio curves baked as custom properties (`kick`, `bass`,
  `mid`, `snare`, `hihat`, `master_energy`). Reference only — not auto-wired to anything.
  User views curves in Graph Editor alongside audio in the Sequencer.
- **PatternOrigin** — Empty used as the texture coordinate object for all pattern materials.
  Move/rotate it to move/rotate all patterns coherently. Per-surface offset via each
  surface mesh's own object origin.

**Standard property interface** on each surface object wearing a pattern material:

| Property | Type | Default | Controls |
|----------|------|---------|----------|
| `brightness` | float | 1.0 | Emission strength (0-10) |
| `hue` | float | 0.5 | HSV hue (0-1 cyclic, drives Hue Saturation Value node) |
| `speed` | float | 0.0 | Pattern scroll speed (drives Mapping offset over time) |
| `flash` | float | 0.0 | Flash intensity (0-1, multiplied into emission) |

Each pattern type adds its own extras (e.g. `line_thickness`, `beam_count`).

**Driver chain**: AudioGuide props → (user keyframes or copies values) → surface object
custom props → drivers → material Value nodes.

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

### Step 2: Extract frequency bands

Run this **inside Blender** (via HTTP). Uses `wave` + `numpy` (bundled).

```python
import bpy, wave, numpy as np

def extract_bands(wav_path, fps, frame_count):
    """Extract frequency band amplitudes per frame using FFT."""
    wf = wave.open(wav_path, 'rb')
    sr = wf.getframerate()
    raw = wf.readframes(wf.getnframes())
    wf.close()
    audio = np.frombuffer(raw, dtype=np.int16).astype(np.float64) / 32768.0
    spf = sr // fps  # samples per frame
    window_size = 2048

    bands = {
        "kick":  (40, 100),     # narrow: pure kick thump, excludes bass synth
        "bass":  (100, 400),
        "mid":   (400, 2000),
        "snare": (2000, 6000),
        "hihat": (6000, 16000),
    }

    # Bands that need transient sharpening (power curve exponent).
    # Higher = more peaky. Kick especially needs this to avoid muddy sustained energy.
    sharpness = {
        "kick": 2.0,   # square — sharpens transients while staying visible
        "snare": 1.5,  # gentle — snare has some sustain
    }

    results = {name: [] for name in bands}
    results["master_energy"] = []

    for i in range(frame_count):
        start = i * spf
        chunk = audio[start:start + window_size]
        if len(chunk) < window_size:
            for name in results:
                results[name].append(0.0)
            continue

        fft = np.fft.rfft(chunk)
        freqs = np.fft.rfftfreq(window_size, 1.0 / sr)
        magnitudes = np.abs(fft)

        energy_sum = 0.0
        for name, (lo, hi) in bands.items():
            mask = (freqs >= lo) & (freqs <= hi)
            val = float(np.mean(magnitudes[mask])) if mask.any() else 0.0
            results[name].append(val)
            energy_sum += val

        results["master_energy"].append(energy_sum / len(bands))

    # Normalize each band to 0-1, then apply sharpness
    for name in results:
        vals = results[name]
        mx = max(vals) if vals else 1.0
        if mx > 0:
            results[name] = [v / mx for v in vals]
        # Apply power curve for transient sharpening
        exp = sharpness.get(name)
        if exp:
            results[name] = [v ** exp for v in results[name]]
            mx2 = max(results[name]) if results[name] else 1.0
            if mx2 > 0:
                results[name] = [v / mx2 for v in results[name]]

    return results
```

### Step 3: Bake to AudioGuide object

```python
import bpy

def bake_audio_guide(bands, frame_start=1):
    """Create AudioGuide empty and bake band curves as keyframes."""
    guide = bpy.data.objects.get("AudioGuide")
    if not guide:
        guide = bpy.data.objects.new("AudioGuide", None)
        bpy.context.scene.collection.objects.link(guide)
        guide.empty_display_type = 'ARROWS'
        guide.empty_display_size = 0.3

    for name, values in bands.items():
        for i, val in enumerate(values):
            guide[name] = val
            guide.keyframe_insert(data_path=f'["{name}"]', frame=frame_start + i)

    # Set LINEAR interpolation for snappy response
    if guide.animation_data and guide.animation_data.action:
        for fc in guide.animation_data.action.fcurves:
            for kp in fc.keyframe_points:
                kp.interpolation = 'LINEAR'

    return guide
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

### Complete extraction workflow

```python
# Outside Blender (shell):
ffmpeg -y -i /path/to/song.ogg -ar 44100 -ac 1 -sample_fmt s16 output/song.wav

# Inside Blender (via HTTP):
wav_path = f"{OUTPUT}/song.wav"
fps = bpy.context.scene.render.fps
frame_count = bpy.context.scene.frame_end - bpy.context.scene.frame_start + 1

bands = extract_bands(wav_path, fps, frame_count)
guide = bake_audio_guide(bands, frame_start=bpy.context.scene.frame_start)

# Load audio for playback
if not bpy.context.scene.sequence_editor:
    bpy.context.scene.sequence_editor_create()
se = bpy.context.scene.sequence_editor
se.strips.new_sound(name="Music", filepath=wav_path, channel=1,
                    frame_start=bpy.context.scene.frame_start)
```

### BPM ramp

The BPM ramp is a linear time signal that, when multiplied by a factor, gives beat-synced
oscillations at different subdivisions. **The user draws this ramp manually** because tempo
may change throughout the song.

**Agent guidance for the user:**
- Create a custom property `bpm_ramp` on AudioGuide (or a separate control object)
- Keyframe it as a linear ramp: value = `beat_number` at each beat
- For a constant-tempo track at 128 BPM: slope = 128/60 = 2.133 beats per second
- In driver expressions, use `bpm_ramp * N` for N-beat subdivision:
  - `sin(bpm_ramp * 2 * pi)` = 1-beat pulse
  - `sin(bpm_ramp * 8 * 2 * pi)` = 8th-note pulse
  - `sin(bpm_ramp * 16 * 2 * pi)` = 16th-note flash

## Control Object Setup

```python
import bpy

# AudioGuide (audio reference curves)
guide = bpy.data.objects.new("AudioGuide", None)
bpy.context.scene.collection.objects.link(guide)
guide.empty_display_type = 'ARROWS'
guide.empty_display_size = 0.3

# PatternOrigin (shared texture coordinate source)
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
| **Radial** | ATAN2 + sine + threshold | `beam_count`, `beam_width` |
| **Concentric** | Distance + sine + threshold | `ring_count`, `ring_width` |
| **Polar** | Normal vector + radial math | `rings`, `spokes` |

See `pattern-reference.md` for complete Python code for each pattern.

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

## Wiring Drivers

### AudioGuide property → surface custom property

Copy values manually (user keyframes) or wire a driver for live preview:

```python
obj = bpy.data.objects["Floor"]
guide = bpy.data.objects["AudioGuide"]

# Drive floor brightness from kick
drv = obj.driver_add('["brightness"]')
var = drv.driver.variables.new()
var.name = "kick"
var.targets[0].id = guide
var.targets[0].data_path = '["kick"]'
drv.driver.expression = "kick * 3.0"  # scale to taste
```

### Surface property → material node

```python
mat = obj.data.materials[0]
nodes = mat.node_tree.nodes

# Drive brightness Value node from object property
bright_node = next(n for n in nodes if n.label == "brightness")
drv = bright_node.outputs[0].driver_add("default_value")
var = drv.driver.variables.new()
var.name = "val"
var.targets[0].id = obj
var.targets[0].data_path = '["brightness"]'
drv.driver.expression = "val"

# Drive HSV hue from object property
hsv_node = next(n for n in nodes if n.label == "color_control")
drv = hsv_node.inputs["Hue"].driver_add("default_value")
var = drv.driver.variables.new()
var.name = "val"
var.targets[0].id = obj
var.targets[0].data_path = '["hue"]'
drv.driver.expression = "val"
```

### Speed (pattern scrolling via frame)

```python
mapping = next(n for n in nodes if n.label == "pattern_mapping")

# Drive Mapping X location from speed * frame
drv = mapping.inputs["Location"].driver_add("default_value", 0)  # index 0 = X
var = drv.driver.variables.new()
var.name = "spd"
var.targets[0].id = obj
var.targets[0].data_path = '["speed"]'
drv.driver.expression = "spd * frame / 60.0"  # adjust 60.0 to scene fps
```

### Flash (emission pulse)

```python
# Flash multiplies into brightness: brightness * (1 + flash * sin(time))
bright_node = next(n for n in nodes if n.label == "brightness")
drv = bright_node.outputs[0].driver_add("default_value")
# Two variables: base brightness + flash
var_b = drv.driver.variables.new()
var_b.name = "bright"
var_b.targets[0].id = obj
var_b.targets[0].data_path = '["brightness"]'
var_f = drv.driver.variables.new()
var_f.name = "flash"
var_f.targets[0].id = obj
var_f.targets[0].data_path = '["flash"]'
drv.driver.expression = "bright * (1.0 + flash * sin(frame * 0.5))"
```

### Wiring to character materials (any material)

To make any existing material audio-reactive, inject a driver on its emission strength:

```python
mat = bpy.data.materials["clothes01"]
bsdf = mat.node_tree.nodes["Principled BSDF"]
guide = bpy.data.objects["AudioGuide"]

drv = bsdf.inputs["Emission Strength"].driver_add("default_value")
var = drv.driver.variables.new()
var.name = "energy"
var.targets[0].id = guide
var.targets[0].data_path = '["master_energy"]'
drv.driver.expression = "energy * 2.0"
```

For bulk application to all materials on a character:

```python
armature = bpy.data.objects["Armature"]
guide = bpy.data.objects["AudioGuide"]

for child in armature.children:
    if child.type != 'MESH':
        continue
    for mat in child.data.materials:
        if not mat or not mat.use_nodes:
            continue
        bsdf = mat.node_tree.nodes.get("Principled BSDF")
        if not bsdf:
            continue
        drv = bsdf.inputs["Emission Strength"].driver_add("default_value")
        var = drv.driver.variables.new()
        var.name = "e"
        var.targets[0].id = guide
        var.targets[0].data_path = '["master_energy"]'
        drv.driver.expression = "e * 2.0"
```

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

## Lighting Rigs

Three presets with a shared interface. See `lighting-rigs.md` for complete setup code.

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
