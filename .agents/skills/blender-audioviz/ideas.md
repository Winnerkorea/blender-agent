# Audio Visualization — Creative Ideas

Advanced techniques for future scenes. Each idea notes which existing skills/tools
it builds on and rough implementation complexity.

## Geometry-driven effects

### Audio-reactive floor displacement
Geometry nodes modifier on the floor that displaces tiles vertically with kick energy.
Creates a "graphic equalizer floor" — tiles physically rise and fall, catching light
from the ring rig. Use Store Named Attribute to pass audio data per-tile.
- **Builds on**: geometry-nodes skill, audioviz control architecture
- **Complexity**: Medium — GN modifier + driver wiring

### Particle burst on beat drops
Geometry nodes setup that emits emissive shards/sparks from the floor when kick
crosses a threshold. Particles fly up, fade emission, disappear. A frame handler
detects threshold crossings and triggers the burst via a custom property.
- **Builds on**: geometry-nodes simulation zones, audioviz frame handler pattern
- **Complexity**: Medium-High — simulation zone + threshold detection

### Ribbon trails on character bones
Track specific bones (wrists, ankles) and spawn curve geometry that trails behind
them. Emission color driven by audio band. Creates flowing light trails like concert
visuals. Use a frame handler to sample bone world positions and build trail meshes.
- **Builds on**: laser skill mesh-building pattern, armature bone tracking
- **Complexity**: High — bone tracking + dynamic mesh generation per frame

## Camera and post-processing

### Beat-synced camera cuts
Auto-generate VSE scene strips that alternate between cameras aligned to the BPM
ramp. Produces a multi-cam music video edit without manual cutting. User provides
BPM ramp + camera list, skill generates the switching points.
- **Builds on**: vse skill, audioviz BPM ramp
- **Complexity**: Low — VSE strip creation from beat markers

### Chromatic aberration on drops
Compositor lens distortion that pulses with kick. Subtle most of the time, but on
big hits the image briefly splits into RGB channels. Wire the distortion amount
to the kick curve via compositor node drivers.
- **Builds on**: blender-3d compositor section
- **Complexity**: Low — compositor node + single driver

### Motion blur ramping
Increase motion blur samples during fast dance sections, reduce during slow parts.
Drive `scene.eevee.motion_blur_shutter` from master_energy curve. Emphasizes
movement during high-energy sections.
- **Builds on**: audioviz driver wiring
- **Complexity**: Low — single property driver

## Lighting and atmosphere

### Silhouette mode
Strong backlight that turns the character into a dark silhouette against bright
wall/floor patterns. Toggle between front-lit and silhouette using a single "mode"
property that crossfades between front and back light energy. Dramatic for
verse-to-chorus transitions.
- **Builds on**: audioviz lighting rigs
- **Complexity**: Low — light energy keyframing

### Lightning/strobe effect
A point light at maximum intensity for exactly 1 frame on snare hits, then off.
Creates a freeze-frame strobe effect during intense sections. Use a frame handler
that checks the snare curve against a threshold and sets light energy accordingly.
- **Builds on**: audioviz frame handler pattern
- **Complexity**: Low — threshold check + light toggle

### Color palette shifts by song section
Define 3-4 named color palettes (cool blues for verse, warm reds for chorus, white
for bridge). A "palette_mix" property crossfades between them. More intentional than
continuous hue cycling — creates distinct visual identity per song section.
- **Builds on**: audioviz HSV color system
- **Complexity**: Medium — palette interpolation in driver expressions or frame handler

### Fog density breathing
Slowly pulse volumetric fog density with bass energy. Low bass = thin fog (sharp
beams visible), high bass = thick fog (diffuse glow). Changes the entire scene mood
per section. Drive `Volume Principled.Density` from bass curve.
- **Builds on**: audioviz driver wiring, blender-3d volumetric section
- **Complexity**: Low — single driver on fog material

## Surface and material

### Reflective floor
Change the floor material to glossy (metallic=1.0, roughness=0.05-0.15). The floor
reflects the character and ceiling grid, creating a "standing on glass" look.
Reflections double all light effects for free. Works with EEVEE screen-space
reflections. Can blend between matte and reflective using a "reflectivity" property.
- **Builds on**: existing floor material
- **Complexity**: Very Low — property change on existing material

### Animated Voronoi/organic patterns
Replace geometric grids with Voronoi texture for organic cell patterns that morph
with audio. Animate the Voronoi W input or Scale for shifting cells. Feels more
biological/alien vs the clean tech grid. Good for contrasting song sections.
- **Builds on**: audioviz pattern system
- **Complexity**: Medium — new pattern type using Voronoi texture node

### UV-scrolling textures on character
Apply a scrolling emissive grid pattern directly to the character's clothes using
UV coordinates. Toon shader + emissive grid lines on the outfit = Tron-style look.
Drive scroll speed from master_energy. The pattern follows the mesh deformation
during dance.
- **Builds on**: audioviz pattern materials (use UV coordinates instead of Object)
- **Complexity**: Medium — UV-based pattern + per-material application

## Scene composition

### Mirror room
Make walls partially reflective (metallic + low roughness) instead of opaque.
Creates infinite-corridor effects with the light beams. Works in EEVEE with
screen-space reflections enabled. Combine with the grid line pattern for a
cyberpunk corridor feel.
- **Builds on**: existing wall materials
- **Complexity**: Low — material property change + EEVEE SSR settings

### Floating geometric objects
Audio-reactive icospheres, tori, or abstract shapes orbiting the character. Each
object driven by a different frequency band: kick = big sphere pulses scale,
hihat = small particles shimmer emission, bass = slow orbit rotation. Use geometry
nodes for instancing or simple object arrays.
- **Builds on**: geometry-nodes skill, audioviz driver wiring
- **Complexity**: Medium — object creation + multi-band driver wiring

### Stage transitions
Floor splits open, walls rotate away, ceiling rises — all keyframed to song
structure. The stage itself physically transforms between song sections. Use
object keyframes on the stage geometry, timed to the BPM ramp or manual markers.
- **Builds on**: audioviz BPM ramp, standard keyframing
- **Complexity**: Medium — choreographed keyframe sequences

## Recommended starting points

Highest impact for least effort:
1. **Reflective floor** — one property change, doubles visual richness
2. **Silhouette mode** — dramatic mood shift with just light keyframing
3. **Fog density breathing** — single driver, transforms atmosphere
4. **Beat-synced camera cuts** — automates tedious manual editing
5. **Chromatic aberration on drops** — small compositor addition, big punch
