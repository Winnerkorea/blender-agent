# Lighting Rigs — Stage Preset Reference

Complete setup code for the three light rig presets. All share the same control
interface and can be driven from AudioGuide properties via drivers.

## Common Setup

### HSV color for lights

Blender light color is RGB, but we want HSV-driven hue cycling. Register a helper
in the driver namespace (once per session):

```python
import bpy, colorsys

def hsv_rgb(hue, sat, val, component):
    """Return R, G, or B (component 0/1/2) from HSV inputs."""
    r, g, b = colorsys.hsv_to_rgb(hue % 1.0, sat, val)
    return [r, g, b][int(component)]

bpy.app.driver_namespace["hsv_rgb"] = hsv_rgb
```

Then drive each light's RGB color channels:
```python
light = bpy.data.objects["Spot_0"]
rig = bpy.data.objects["LightRig"]

for i, channel in enumerate(["color"]):  # light.data.color is a 3-component vector
    for comp in range(3):
        drv = light.data.driver_add("color", comp)
        var_h = drv.driver.variables.new()
        var_h.name = "hue"
        var_h.targets[0].id = rig
        var_h.targets[0].data_path = '["hue"]'
        var_s = drv.driver.variables.new()
        var_s.name = "sat"
        var_s.targets[0].id = rig
        var_s.targets[0].data_path = '["saturation"]'
        drv.driver.expression = f"hsv_rgb(hue, sat, 1.0, {comp})"
```

### Control object

All rigs create a `LightRig` control empty with shared properties:

```python
def create_rig_controls():
    """Create the LightRig control object with standard properties."""
    rig = bpy.data.objects.new("LightRig", None)
    bpy.context.scene.collection.objects.link(rig)
    rig.empty_display_type = 'CIRCLE'
    rig.empty_display_size = 0.5

    rig["count"] = 8
    rig["radius"] = 3.0
    rig["focus"] = 1.0       # positive=converge, negative=diverge
    rig["energy"] = 3000.0
    rig["hue"] = 0.6         # 0-1 cyclic
    rig["saturation"] = 1.0  # 0-1
    rig["sweep_angle"] = 0.0 # rotation offset (radians, keyframable)

    return rig
```

## Ring Rig

N spot lights evenly spaced on a circle, all aiming at a shared target.

```python
import bpy, math

def create_ring_rig(count=8, radius=3.0, height=5.0, center=(0, 0, 0)):
    """Create a ring of spot lights with shared target control."""
    rig = create_rig_controls()
    rig.location = center

    # Target empty at center (beams converge here)
    target = bpy.data.objects.new("RigTarget", None)
    bpy.context.scene.collection.objects.link(target)
    target.empty_display_type = 'SPHERE'
    target.empty_display_size = 0.3
    target.location = (center[0], center[1], center[2] + 0.5)
    target.parent = rig

    # Create spots
    spots = []
    for i in range(count):
        angle = (2 * math.pi * i) / count
        x = center[0] + radius * math.cos(angle)
        y = center[1] + radius * math.sin(angle)
        z = center[2] + height

        bpy.ops.object.light_add(type='SPOT', location=(x, y, z))
        spot = bpy.context.active_object
        spot.name = f"RingSpot_{i}"
        spot.data.energy = 3000
        spot.data.spot_size = math.radians(45)
        spot.data.spot_blend = 0.1
        spot.data.shadow_soft_size = 0.05
        spot.data.use_shadow = True

        # Track-To target
        track = spot.constraints.new(type='TRACK_TO')
        track.target = target
        track.track_axis = 'TRACK_NEGATIVE_Z'
        track.up_axis = 'UP_Y'

        # Store radial direction for focus/diverge calculations
        spot["rig_angle"] = angle
        spot["rig_index"] = i

        spots.append(spot)

    # Drive energy from rig control
    for spot in spots:
        drv = spot.data.driver_add("energy")
        var = drv.driver.variables.new()
        var.name = "e"
        var.targets[0].id = rig
        var.targets[0].data_path = '["energy"]'
        drv.driver.expression = "e"

    return rig, target, spots
```

### Focus parameter

Focus controls beam convergence/divergence by moving per-light targets:

```python
def wire_focus(rig, target, spots):
    """Wire focus parameter: positive = converge at center, negative = fan outward.

    Implementation: each spot gets its own target offset driven by focus.
    When focus > 0, all targets stay at center. When focus < 0, targets
    push outward along each spot's radial direction.
    """
    for spot in spots:
        angle = spot["rig_angle"]
        # For each spot, create a child target that offsets radially
        child_target = bpy.data.objects.new(f"Target_{spot.name}", None)
        bpy.context.scene.collection.objects.link(child_target)
        child_target.parent = target
        child_target.empty_display_size = 0.1

        # Update Track-To to use individual target
        spot.constraints["Track To"].target = child_target

        # Drive child target X offset: when focus < 0, push outward
        import math
        cos_a = math.cos(angle)
        sin_a = math.sin(angle)

        drv_x = child_target.driver_add("location", 0)
        var = drv_x.driver.variables.new()
        var.name = "f"
        var.targets[0].id = rig
        var.targets[0].data_path = '["focus"]'
        drv_x.driver.expression = f"min(0, f) * {-cos_a * 3:.4f}"

        drv_y = child_target.driver_add("location", 1)
        var = drv_y.driver.variables.new()
        var.name = "f"
        var.targets[0].id = rig
        var.targets[0].data_path = '["focus"]'
        drv_y.driver.expression = f"min(0, f) * {-sin_a * 3:.4f}"
```

### Sweep animation

Rotate the entire rig for sweeping beam effects:

```python
# Keyframe sweep_angle on rig, then drive rig rotation
rig = bpy.data.objects["LightRig"]
drv = rig.driver_add("rotation_euler", 2)  # Z rotation
var = drv.driver.variables.new()
var.name = "sweep"
var.targets[0].id = rig
var.targets[0].data_path = '["sweep_angle"]'
drv.driver.expression = "sweep"

# Or drive from audio for reactive sweeping:
# drv.driver.expression = "sweep + kick * 0.3"
```

## Cluster Rig

N spots at a single point with angular offsets, creating a concentrated beam bundle.

```python
import bpy, math

def create_cluster_rig(count=6, spread=0.15, height=5.0, center=(0, 0, 0)):
    """Create a cluster of spots at one point with slight angular spread."""
    rig = create_rig_controls()
    rig.location = center
    rig["radius"] = spread  # repurpose radius as angular spread

    target = bpy.data.objects.new("ClusterTarget", None)
    bpy.context.scene.collection.objects.link(target)
    target.empty_display_type = 'SPHERE'
    target.empty_display_size = 0.3
    target.location = (center[0], center[1], 0)
    target.parent = rig

    spots = []
    for i in range(count):
        angle = (2 * math.pi * i) / count
        # Small offset from center point
        x = center[0] + spread * math.cos(angle)
        y = center[1] + spread * math.sin(angle)
        z = center[2] + height

        bpy.ops.object.light_add(type='SPOT', location=(x, y, z))
        spot = bpy.context.active_object
        spot.name = f"ClusterSpot_{i}"
        spot.data.energy = 3000 / count  # divide energy
        spot.data.spot_size = math.radians(30)
        spot.data.spot_blend = 0.15
        spot.data.shadow_soft_size = 0.05
        spot.data.use_shadow = True

        track = spot.constraints.new(type='TRACK_TO')
        track.target = target
        track.track_axis = 'TRACK_NEGATIVE_Z'
        track.up_axis = 'UP_Y'

        spot["rig_angle"] = angle
        spots.append(spot)

    return rig, target, spots

# Focus works the same way as ring rig — wire_focus() applies identically
```

## Disco Ball

Fake disco ball using a faceted icosphere with emissive material + orbiting small spots.

```python
import bpy, math

def create_disco_ball(location=(0, 0, 4), ball_radius=0.3, spot_count=6,
                      spot_orbit_radius=2.0):
    """Create a fake disco ball with orbiting spots for scattered light effect."""
    rig = create_rig_controls()
    rig.location = location

    # --- Faceted ball (visual only, emissive facets) ---
    bpy.ops.mesh.primitive_ico_sphere_add(
        subdivisions=2, radius=ball_radius, location=location
    )
    ball = bpy.context.active_object
    ball.name = "DiscoBall"
    ball.parent = rig

    # Flat shading for faceted look
    for poly in ball.data.polygons:
        poly.use_smooth = False

    # Emissive facet material
    mat = bpy.data.materials.new("DiscoBallMat")
    mat.use_nodes = True
    nodes = mat.node_tree.nodes
    bsdf = nodes["Principled BSDF"]
    bsdf.inputs["Metallic"].default_value = 1.0
    bsdf.inputs["Roughness"].default_value = 0.05
    bsdf.inputs["Emission Strength"].default_value = 2.0
    bsdf.inputs["Emission Color"].default_value = (1, 1, 1, 1)
    ball.data.materials.append(mat)

    # --- Orbiting small spots (fake reflections) ---
    spots = []
    for i in range(spot_count):
        angle = (2 * math.pi * i) / spot_count
        x = location[0] + spot_orbit_radius * math.cos(angle)
        y = location[1] + spot_orbit_radius * math.sin(angle)
        z = location[2]

        bpy.ops.object.light_add(type='SPOT', location=(x, y, z))
        spot = bpy.context.active_object
        spot.name = f"DiscoSpot_{i}"
        spot.data.energy = 500
        spot.data.spot_size = math.radians(15)  # narrow beam
        spot.data.spot_blend = 0.3
        spot.data.shadow_soft_size = 0.02
        spot.data.use_shadow = True

        # Parent to rig so they orbit when rig rotates
        spot.parent = rig

        # Aim downward and slightly outward
        spot.rotation_euler = (math.radians(60 + i * 10), 0, angle)

        spots.append(spot)

    return rig, ball, spots
```

### Spinning animation

```python
rig = bpy.data.objects["LightRig"]

# Constant rotation
rig.rotation_euler = (0, 0, 0)
rig.keyframe_insert(data_path="rotation_euler", index=2, frame=1)
rig.rotation_euler.z = math.pi * 2
rig.keyframe_insert(data_path="rotation_euler", index=2, frame=120)

# Linear interpolation for constant speed
for fc in rig.animation_data.action.fcurves:
    for kp in fc.keyframe_points:
        kp.interpolation = 'LINEAR'
```

### Audio-driven disco

```python
# Drive rotation speed from master_energy
guide = bpy.data.objects["AudioGuide"]
rig = bpy.data.objects["LightRig"]

drv = rig.driver_add("rotation_euler", 2)
var = drv.driver.variables.new()
var.name = "e"
var.targets[0].id = guide
var.targets[0].data_path = '["master_energy"]'
drv.driver.expression = "e * frame * 0.02"  # speed scales with energy

# Pulse ball emission from kick
ball_mat = bpy.data.materials["DiscoBallMat"]
bsdf = ball_mat.node_tree.nodes["Principled BSDF"]
drv = bsdf.inputs["Emission Strength"].driver_add("default_value")
var = drv.driver.variables.new()
var.name = "kick"
var.targets[0].id = guide
var.targets[0].data_path = '["kick"]'
drv.driver.expression = "1.0 + kick * 5.0"
```

## Wiring Audio to Any Rig

All three rigs share the same property interface. Wire from AudioGuide:

```python
rig = bpy.data.objects["LightRig"]
guide = bpy.data.objects["AudioGuide"]

# Energy from kick
drv = rig.driver_add('["energy"]')
var = drv.driver.variables.new()
var.name = "kick"
var.targets[0].id = guide
var.targets[0].data_path = '["kick"]'
drv.driver.expression = "1000 + kick * 5000"

# Hue cycling from master_energy
drv = rig.driver_add('["hue"]')
var = drv.driver.variables.new()
var.name = "e"
var.targets[0].id = guide
var.targets[0].data_path = '["master_energy"]'
drv.driver.expression = "e * 0.5 + frame * 0.002"  # slow drift + energy modulation

# Focus pulsing from snare
drv = rig.driver_add('["focus"]')
var = drv.driver.variables.new()
var.name = "snare"
var.targets[0].id = guide
var.targets[0].data_path = '["snare"]'
drv.driver.expression = "1.0 - snare * 3.0"  # converge normally, fan out on snare hits
```
