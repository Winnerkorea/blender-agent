# Pattern Reference — Complete Material Recipes

Full Python code for each pattern type. All patterns use `create_pattern_base()` from
the main SKILL.md, then add pattern-specific math chains.

## Grid Pattern

Brick Texture-based grid with moving, flashing, rotating lines. Closest to the
current "tiles" material. Two-color system with independent line and tile brightness.

```python
import bpy, math

def create_grid_pattern(obj, origin_empty):
    """Grid pattern using Brick Texture. Supports move, rotate, flash, 2-color blend."""
    mat, nodes, links, sep, hsv, bright_val, mapping = create_pattern_base(
        "GridPattern", obj, origin_empty
    )

    # Pattern-specific controls
    obj["line_thickness"] = 0.02   # mortar size (0.005-0.1)
    obj["tile_size"] = 0.4         # brick texture scale
    obj["color_mix"] = 0.0         # blend between color1 and color2 (-1 to 1)

    # Brick Texture
    brick = nodes.new("ShaderNodeTexBrick")
    brick.location = (0, 200)
    brick.inputs["Scale"].default_value = 0.4
    brick.inputs["Mortar Size"].default_value = 2.0
    brick.inputs["Mortar Smooth"].default_value = 1.0
    brick.inputs["Brick Width"].default_value = 1.0
    brick.inputs["Row Height"].default_value = 1.0
    links.new(brick.inputs["Vector"], mapping.outputs["Vector"])

    # Brick Color1 and Color2 (driven by hue + color_mix)
    brick.inputs["Color1"].default_value = (0.0, 0.5, 1.0, 1)
    brick.inputs["Color2"].default_value = (0.0, 1.0, 0.5, 1)
    brick.inputs["Mortar"].default_value = (0, 0, 0, 1)  # black mortar

    # The Brick "Color" output gives tile color, "Fac" gives mortar mask
    # Use Fac as the line detection: Fac=0 means mortar (line), Fac=1 means brick
    # Invert: lines are where Fac is LOW
    invert = nodes.new("ShaderNodeMath")
    invert.operation = 'LESS_THAN'
    invert.inputs[1].default_value = 0.5  # threshold
    links.new(invert.inputs[0], brick.outputs["Fac"])

    # Mix: mortar (lines) use emission, bricks use base
    mix = nodes.new("ShaderNodeMix")
    mix.data_type = 'RGBA'
    mix.location = (150, 100)
    mix.inputs["A"].default_value = (0.02, 0.02, 0.02, 1)  # tile base (dark)
    links.new(mix.inputs["Factor"], invert.outputs[0])
    links.new(hsv.inputs["Color"], mix.outputs["Result"])

    # Line color from Brick Color output → Mix B input
    links.new(mix.inputs["B"], brick.outputs["Color"])

    # Wire drivers for tile_size → Brick Scale
    drv = brick.inputs["Scale"].driver_add("default_value")
    var = drv.driver.variables.new()
    var.name = "val"
    var.targets[0].id = obj
    var.targets[0].data_path = '["tile_size"]'
    drv.driver.expression = "val"

    # Wire line_thickness → Mortar Size
    drv = brick.inputs["Mortar Size"].driver_add("default_value")
    var = drv.driver.variables.new()
    var.name = "val"
    var.targets[0].id = obj
    var.targets[0].data_path = '["line_thickness"]'
    drv.driver.expression = "val * 100"  # scale to Brick Texture range

    return mat
```

## Checkerboard Pattern

Alternating cells using modulo math.

```python
def create_checkerboard_pattern(obj, origin_empty):
    """Checkerboard pattern — alternating emissive/dark cells."""
    mat, nodes, links, sep, hsv, bright_val, mapping = create_pattern_base(
        "CheckerboardPattern", obj, origin_empty
    )

    obj["cell_size"] = 1.0
    obj["contrast"] = 1.0   # 0 = uniform, 1 = full checker

    mapping.inputs["Scale"].default_value = (4, 4, 1)

    # Floor(X) + Floor(Y) mod 2
    floor_x = nodes.new("ShaderNodeMath")
    floor_x.operation = 'FLOOR'
    links.new(floor_x.inputs[0], sep.outputs["X"])

    floor_y = nodes.new("ShaderNodeMath")
    floor_y.operation = 'FLOOR'
    links.new(floor_y.inputs[0], sep.outputs["Y"])

    add = nodes.new("ShaderNodeMath")
    add.operation = 'ADD'
    links.new(add.inputs[0], floor_x.outputs[0])
    links.new(add.inputs[1], floor_y.outputs[0])

    mod2 = nodes.new("ShaderNodeMath")
    mod2.operation = 'MODULO'
    mod2.inputs[1].default_value = 2.0
    links.new(mod2.inputs[0], add.outputs[0])

    # Mask: mod2 rounds to 0 or 1
    mask = nodes.new("ShaderNodeMath")
    mask.operation = 'GREATER_THAN'
    mask.inputs[1].default_value = 0.5
    links.new(mask.inputs[0], mod2.outputs[0])

    # Mix
    mix = nodes.new("ShaderNodeMix")
    mix.data_type = 'RGBA'
    mix.inputs["A"].default_value = (0.02, 0.02, 0.02, 1)
    mix.inputs["B"].default_value = (0, 0.5, 1.0, 1)
    links.new(mix.inputs["Factor"], mask.outputs[0])
    links.new(hsv.inputs["Color"], mix.outputs["Result"])

    return mat
```

## Hexagonal Pattern

Hex grid using offset rows and distance-to-center math.

```python
def create_hex_pattern(obj, origin_empty):
    """Hexagonal cell pattern with emissive borders."""
    mat, nodes, links, sep, hsv, bright_val, mapping = create_pattern_base(
        "HexPattern", obj, origin_empty
    )

    obj["cell_size"] = 1.0
    obj["border_width"] = 0.05

    mapping.inputs["Scale"].default_value = (4, 4, 1)

    # Hex grid: offset every other row by 0.5
    # row = floor(Y * sqrt(3))
    sqrt3 = nodes.new("ShaderNodeMath")
    sqrt3.operation = 'MULTIPLY'
    sqrt3.inputs[1].default_value = 1.732  # sqrt(3)
    links.new(sqrt3.inputs[0], sep.outputs["Y"])

    row = nodes.new("ShaderNodeMath")
    row.operation = 'FLOOR'
    links.new(row.inputs[0], sqrt3.outputs[0])

    # Offset X by 0.5 on odd rows
    row_mod = nodes.new("ShaderNodeMath")
    row_mod.operation = 'MODULO'
    row_mod.inputs[1].default_value = 2.0
    links.new(row_mod.inputs[0], row.outputs[0])

    offset = nodes.new("ShaderNodeMath")
    offset.operation = 'MULTIPLY'
    offset.inputs[1].default_value = 0.5
    links.new(offset.inputs[0], row_mod.outputs[0])

    x_offset = nodes.new("ShaderNodeMath")
    x_offset.operation = 'ADD'
    links.new(x_offset.inputs[0], sep.outputs["X"])
    links.new(x_offset.inputs[1], offset.outputs[0])

    # Cell center: fract(adjusted_x) - 0.5, fract(adjusted_y) - 0.5
    frac_x = nodes.new("ShaderNodeMath")
    frac_x.operation = 'FRACT'
    links.new(frac_x.inputs[0], x_offset.outputs[0])

    sub_x = nodes.new("ShaderNodeMath")
    sub_x.operation = 'SUBTRACT'
    sub_x.inputs[1].default_value = 0.5
    links.new(sub_x.inputs[0], frac_x.outputs[0])

    frac_y = nodes.new("ShaderNodeMath")
    frac_y.operation = 'FRACT'
    links.new(frac_y.inputs[0], sqrt3.outputs[0])

    sub_y = nodes.new("ShaderNodeMath")
    sub_y.operation = 'SUBTRACT'
    sub_y.inputs[1].default_value = 0.5
    links.new(sub_y.inputs[0], frac_y.outputs[0])

    # Distance from cell center
    sq_x = nodes.new("ShaderNodeMath")
    sq_x.operation = 'POWER'
    sq_x.inputs[1].default_value = 2.0
    links.new(sq_x.inputs[0], sub_x.outputs[0])

    sq_y = nodes.new("ShaderNodeMath")
    sq_y.operation = 'POWER'
    sq_y.inputs[1].default_value = 2.0
    links.new(sq_y.inputs[0], sub_y.outputs[0])

    dist_sq = nodes.new("ShaderNodeMath")
    dist_sq.operation = 'ADD'
    links.new(dist_sq.inputs[0], sq_x.outputs[0])
    links.new(dist_sq.inputs[1], sq_y.outputs[0])

    dist = nodes.new("ShaderNodeMath")
    dist.operation = 'SQRT'
    links.new(dist.inputs[0], dist_sq.outputs[0])

    # Border mask: distance > (0.5 - border_width)
    border = nodes.new("ShaderNodeMath")
    border.operation = 'GREATER_THAN'
    border.inputs[1].default_value = 0.4  # 0.5 - border_width
    border.label = "border_threshold"
    links.new(border.inputs[0], dist.outputs[0])

    # Mix
    mix = nodes.new("ShaderNodeMix")
    mix.data_type = 'RGBA'
    mix.inputs["A"].default_value = (0.02, 0.02, 0.02, 1)
    mix.inputs["B"].default_value = (0, 0.5, 1.0, 1)
    links.new(mix.inputs["Factor"], border.outputs[0])
    links.new(hsv.inputs["Color"], mix.outputs["Result"])

    return mat
```

## Radial Pattern (Spokes)

Lines radiating from center, like a starburst.

```python
def create_radial_pattern(obj, origin_empty):
    """Radial spoke pattern — lines radiating from center."""
    mat, nodes, links, sep, hsv, bright_val, mapping = create_pattern_base(
        "RadialPattern", obj, origin_empty
    )

    obj["beam_count"] = 12
    obj["beam_width"] = 0.97  # threshold: 0.9=thick, 0.99=hair-thin

    # Angle from center: atan2(Y, X)
    atan2 = nodes.new("ShaderNodeMath")
    atan2.operation = 'ARCTAN2'
    links.new(atan2.inputs[0], sep.outputs["Y"])
    links.new(atan2.inputs[1], sep.outputs["X"])

    # Scale by beam count
    mult = nodes.new("ShaderNodeMath")
    mult.operation = 'MULTIPLY'
    mult.inputs[1].default_value = 12.0
    mult.label = "beam_count"
    links.new(mult.inputs[0], atan2.outputs[0])

    # sin → abs → threshold
    sine = nodes.new("ShaderNodeMath")
    sine.operation = 'SINE'
    links.new(sine.inputs[0], mult.outputs[0])

    abs_node = nodes.new("ShaderNodeMath")
    abs_node.operation = 'ABSOLUTE'
    links.new(abs_node.inputs[0], sine.outputs[0])

    gt = nodes.new("ShaderNodeMath")
    gt.operation = 'GREATER_THAN'
    gt.inputs[1].default_value = 0.97
    gt.label = "beam_width"
    links.new(gt.inputs[0], abs_node.outputs[0])

    # Mix
    mix = nodes.new("ShaderNodeMix")
    mix.data_type = 'RGBA'
    mix.inputs["A"].default_value = (0.02, 0.02, 0.02, 1)
    mix.inputs["B"].default_value = (0, 0.5, 1.0, 1)
    links.new(mix.inputs["Factor"], gt.outputs[0])
    links.new(hsv.inputs["Color"], mix.outputs["Result"])

    return mat
```

## Concentric Rings Pattern

Rings expanding from center.

```python
def create_concentric_pattern(obj, origin_empty):
    """Concentric rings pattern — rings expanding from center."""
    mat, nodes, links, sep, hsv, bright_val, mapping = create_pattern_base(
        "ConcentricPattern", obj, origin_empty
    )

    obj["ring_count"] = 8.0    # ring density
    obj["ring_width"] = 0.95   # threshold: 0.85=thick, 0.99=hair-thin

    # Distance from center
    pow_x = nodes.new("ShaderNodeMath")
    pow_x.operation = 'POWER'
    pow_x.inputs[1].default_value = 2.0
    links.new(pow_x.inputs[0], sep.outputs["X"])

    pow_y = nodes.new("ShaderNodeMath")
    pow_y.operation = 'POWER'
    pow_y.inputs[1].default_value = 2.0
    links.new(pow_y.inputs[0], sep.outputs["Y"])

    add = nodes.new("ShaderNodeMath")
    add.operation = 'ADD'
    links.new(add.inputs[0], pow_x.outputs[0])
    links.new(add.inputs[1], pow_y.outputs[0])

    dist = nodes.new("ShaderNodeMath")
    dist.operation = 'SQRT'
    links.new(dist.inputs[0], add.outputs[0])

    # Multiply by ring count and 2pi
    ring_freq = nodes.new("ShaderNodeMath")
    ring_freq.operation = 'MULTIPLY'
    ring_freq.inputs[1].default_value = 8.0
    ring_freq.label = "ring_count"
    links.new(ring_freq.inputs[0], dist.outputs[0])

    mult_2pi = nodes.new("ShaderNodeMath")
    mult_2pi.operation = 'MULTIPLY'
    mult_2pi.inputs[1].default_value = 6.2832  # 2 * pi
    links.new(mult_2pi.inputs[0], ring_freq.outputs[0])

    sine = nodes.new("ShaderNodeMath")
    sine.operation = 'SINE'
    links.new(sine.inputs[0], mult_2pi.outputs[0])

    abs_node = nodes.new("ShaderNodeMath")
    abs_node.operation = 'ABSOLUTE'
    links.new(abs_node.inputs[0], sine.outputs[0])

    gt = nodes.new("ShaderNodeMath")
    gt.operation = 'GREATER_THAN'
    gt.inputs[1].default_value = 0.95
    gt.label = "ring_width"
    links.new(gt.inputs[0], abs_node.outputs[0])

    # Mix
    mix = nodes.new("ShaderNodeMix")
    mix.data_type = 'RGBA'
    mix.inputs["A"].default_value = (0.02, 0.02, 0.02, 1)
    mix.inputs["B"].default_value = (0, 0.5, 1.0, 1)
    links.new(mix.inputs["Factor"], gt.outputs[0])
    links.new(hsv.inputs["Color"], mix.outputs["Result"])

    return mat
```

## Polar Grid Pattern (for spheres/cylinders)

Uses Normal vector coordinates instead of Object coordinates.

```python
def create_polar_pattern(obj, origin_empty):
    """Polar grid for spheres — radial lines + concentric rings using Normal coords."""
    mat, nodes, links, sep, hsv, bright_val, mapping = create_pattern_base(
        "PolarPattern", obj, origin_empty, use_normal=True
    )

    obj["rings"] = 6
    obj["spokes"] = 12

    # --- Radial lines (spokes) ---
    atan2 = nodes.new("ShaderNodeMath")
    atan2.operation = 'ARCTAN2'
    links.new(atan2.inputs[0], sep.outputs["Y"])
    links.new(atan2.inputs[1], sep.outputs["X"])

    spoke_mult = nodes.new("ShaderNodeMath")
    spoke_mult.operation = 'MULTIPLY'
    spoke_mult.inputs[1].default_value = 12.0
    spoke_mult.label = "spoke_count"
    links.new(spoke_mult.inputs[0], atan2.outputs[0])

    spoke_sin = nodes.new("ShaderNodeMath")
    spoke_sin.operation = 'SINE'
    links.new(spoke_sin.inputs[0], spoke_mult.outputs[0])

    spoke_abs = nodes.new("ShaderNodeMath")
    spoke_abs.operation = 'ABSOLUTE'
    links.new(spoke_abs.inputs[0], spoke_sin.outputs[0])

    spoke_mask = nodes.new("ShaderNodeMath")
    spoke_mask.operation = 'GREATER_THAN'
    spoke_mask.inputs[1].default_value = 0.97
    links.new(spoke_mask.inputs[0], spoke_abs.outputs[0])

    # --- Concentric rings ---
    pow_x = nodes.new("ShaderNodeMath")
    pow_x.operation = 'POWER'
    pow_x.inputs[1].default_value = 2.0
    links.new(pow_x.inputs[0], sep.outputs["X"])

    pow_y = nodes.new("ShaderNodeMath")
    pow_y.operation = 'POWER'
    pow_y.inputs[1].default_value = 2.0
    links.new(pow_y.inputs[0], sep.outputs["Y"])

    add = nodes.new("ShaderNodeMath")
    add.operation = 'ADD'
    links.new(add.inputs[0], pow_x.outputs[0])
    links.new(add.inputs[1], pow_y.outputs[0])

    dist = nodes.new("ShaderNodeMath")
    dist.operation = 'SQRT'
    links.new(dist.inputs[0], add.outputs[0])

    ring_freq = nodes.new("ShaderNodeMath")
    ring_freq.operation = 'MULTIPLY'
    ring_freq.inputs[1].default_value = 37.7  # 6 * 2pi
    ring_freq.label = "ring_freq"
    links.new(ring_freq.inputs[0], dist.outputs[0])

    ring_sin = nodes.new("ShaderNodeMath")
    ring_sin.operation = 'SINE'
    links.new(ring_sin.inputs[0], ring_freq.outputs[0])

    ring_abs = nodes.new("ShaderNodeMath")
    ring_abs.operation = 'ABSOLUTE'
    links.new(ring_abs.inputs[0], ring_sin.outputs[0])

    ring_mask = nodes.new("ShaderNodeMath")
    ring_mask.operation = 'GREATER_THAN'
    ring_mask.inputs[1].default_value = 0.95
    links.new(ring_mask.inputs[0], ring_abs.outputs[0])

    # --- Combine: spokes OR rings ---
    combine = nodes.new("ShaderNodeMath")
    combine.operation = 'MAXIMUM'
    links.new(combine.inputs[0], spoke_mask.outputs[0])
    links.new(combine.inputs[1], ring_mask.outputs[0])

    # Mix
    mix = nodes.new("ShaderNodeMix")
    mix.data_type = 'RGBA'
    mix.inputs["A"].default_value = (0.02, 0.02, 0.02, 1)
    mix.inputs["B"].default_value = (0, 0.5, 1.0, 1)
    links.new(mix.inputs["Factor"], combine.outputs[0])
    links.new(hsv.inputs["Color"], mix.outputs["Result"])

    return mat
```

## Combining patterns

Overlay two patterns on the same surface using Maximum (union) or Multiply (intersection):

```python
# After creating two separate masks:
combine = nodes.new("ShaderNodeMath")
combine.operation = 'MAXIMUM'    # MAXIMUM = union, MINIMUM = intersection
links.new(combine.inputs[0], mask_a.outputs[0])
links.new(combine.inputs[1], mask_b.outputs[0])
links.new(mix.inputs["Factor"], combine.outputs[0])
```
