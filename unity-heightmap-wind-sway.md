# Unity Shader Graph: Heightmap Wind Sway

A guide to implementing a wind-swaying heightmap in Unity Shader Graph. The heightmap drives vertex displacement on the Y axis, while a time-based sine wave offsets UV sampling to simulate wind. The key insight is sampling the heightmap at an animated UV offset, then pushing vertices based on that value — with more sway at the tip of the mesh than the base.

---

## Step 1 — Set Up the Graph

Create a new **Lit Shader Graph**. On the graph itself, enable the **Vertex** stage output — you'll be writing to *Position* in object space.

---

## Step 2 — Animate the UV (Wind Offset)

This creates the scrolling/rippling motion across the surface.

1. Add a **Time** node → multiply by a **Wind Speed** float property (try `0.5–1.5`)
2. Feed that into a **Vector2** to create a UV offset (plug into X, leave Y at `0` for horizontal wind)
3. Take your mesh's **UV0** and add the animated offset using **Tiling and Offset**, or a simple **Add**
4. This becomes your *animated UV* for sampling the heightmap

---

## Step 3 — Sample the Heightmap

1. Add a **Texture2D** property (your heightmap) → connect to a **Sample Texture 2D** node
2. Plug your animated UV into the **UV** input of the sampler
3. Take the **R** channel output (greyscale heightmap — R = G = B)

---

## Step 4 — Wind Mask by Height *(crucial)*

Without this, the base of grass or plants slides unrealistically. The tip should sway; the root should stay planted.

1. Add a **Position** node set to **Object** space → take the **Y** component
2. **Remap** or **Saturate** it so `Y=0 → 0` (no sway) and `Y=1 → 1` (full sway)
3. **Multiply** this mask by your heightmap sample from Step 3

---

## Step 5 — Scale and Apply Displacement

1. Multiply the masked height by a **Wind Strength** float property (try `0.05–0.3`)
2. Create a **Vector3**: plug displacement into **X** (or both X and Z for diagonal sway), zero for Y and Z
3. **Add** this to the **Position** node in **Object** space
4. Feed the result into the **Vertex Position** output on the Master Stack

---

## Step 6 — Add Vertex Noise *(optional but recommended)*

Pure sine-wave wind looks mechanical. Breaking it up adds organic turbulence.

1. Add a **Simple Noise** or **Gradient Noise** node sampled at animated UVs (use a slower speed and different scale than the main wind)
2. Multiply by a small value (~`0.02`)
3. Add to your displacement Vector3

---

## Recommended Properties

| Property | Type | Default |
|---|---|---|
| Heightmap | Texture2D | — |
| Wind Speed | Float | 0.8 |
| Wind Strength | Float | 0.12 |
| Wind Direction | Vector2 | (1, 0) |
| Noise Scale | Float | 2.0 |

> **Wind Direction tip:** Multiply your `Time × Speed` scalar by the normalised **Wind Direction** Vector2 before adding to UV. This lets you control the wind angle directly from the Inspector.

---

## Common Pitfalls

**Banding or snapping displacement**
Your heightmap likely has lossy compression. Import it with compression disabled or set to *High Quality* — lossy formats destroy the smooth gradient needed for clean displacement.

**The base of the mesh slides around**
You skipped the Y-mask in Step 4. Always multiply displacement by a height-from-base factor so vertices at `Y=0` receive zero influence.

**Sway looks too uniform or robotic**
Add the noise layer from Step 6, or sum two sine waves at different frequencies. A slight random per-instance phase offset (via **Object ID** or a small random float property) also helps crowds of plants look independent.

**Looks wrong on LODs**
The vertex shader runs per-vertex, so low-poly LODs sway stiffly with visible faceting. Ensure your grass and foliage meshes have enough vertical edge loops to carry the displacement smoothly across each LOD level.

---

## Note on Terrain Detail Grass

Unity's built-in grass renderer bypasses Shader Graph for the *Detail Mesh* system. To use this shader with grass, either:

- Use a **custom mesh + material** with this shader assigned directly, or
- Render via `DrawMeshInstanced` from a script, passing a `_Time`-equivalent uniform manually
