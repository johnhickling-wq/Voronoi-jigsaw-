# Skybox + Tiled-Sphere Implementation Notes

Lessons distilled from `panorama-viewer.html` (Better Day prototype). Use as a
checklist when starting a new demo with Skybox AI panoramas + tile tessellation
(geodesic, Voronoi, or otherwise). Three.js r128, single-file HTML.

---

## 1. Skybox / equirectangular pipeline

- Skybox AI exports **equirectangular** images (2:1 aspect, e.g. 2048×1024).
- View from **inside** the sphere, not outside:
  - Camera at the origin (`camera.position.set(0,0,0)`).
  - Materials use `side: THREE.BackSide` so the inward-facing triangles are
    rendered.
- **Texture wrapping**: `wrapS = THREE.RepeatWrapping`, `wrapT =
  THREE.ClampToEdgeWrapping`. RepeatWrapping on S is what lets the seam-fix
  below (UV > 1.0) actually sample correctly.

## 2. UV mapping (the part everyone gets wrong)

```js
function s2uv(x, y, z) {
  return [
    (Math.atan2(z, x) / (2 * Math.PI) + 0.5),
    Math.asin(Math.max(-1, Math.min(1, y))) / Math.PI + 0.5
  ];
}
```

Three rules, learned the hard way:

1. **V is `lat/π + 0.5`, NOT `0.5 - lat/π`.** The other form inverts the image
   vertically.
2. **Negate X** when sampling for inside-sphere view: `s2uv(-x, y, z)`. Without
   this the panorama appears mirrored.
3. **Seam wrap fix.** A triangle whose three UVs straddle the 0/1 seam will
   sample across the whole texture and produce a stretched smear. After
   computing the three UVs of a triangle:
   ```js
   if (Math.max(ua[0], ub[0], uc[0]) - Math.min(...) > 0.5) {
     if (ua[0] < 0.5) ua[0] += 1;  // shift the small ones up by 1
     if (ub[0] < 0.5) ub[0] += 1;
     if (uc[0] < 0.5) uc[0] += 1;
   }
   ```
   Apply this **per triangle**, not per face.

## 3. Tile triangulation matters

Don't render each tile as a single n-gon — texture interpolation across a flat
n-gon does not match the spherical surface, causing visible warping inside the
tile. Instead:

- Subdivide each tile into a fan from its center to each pair of adjacent
  vertices, then subdivide each fan slice into rows.
- For Better Day: `fS = max(2, ceil(8 / subdivLevel))` rows per fan slice.
- Recompute UVs with `s2uv(...)` at every interpolated vertex (not just at the
  tile's outer corners).
- Apply the seam-wrap fix at every triangle.

This is what makes textures look continuous across the curved tile.

## 4. Tile previews — render to texture, not UV math

Multiple manual approaches to "show a single tile" failed:
- Sampling the equirectangular texture by UV bounds: wrong content at seams,
  black tiles, distortion at poles.
- Rendering the tile's content onto another tile's mesh: UVs are
  geometry-specific, produces distortion.

**The pattern that works:**

1. Hide every face mesh except the target.
2. Set the target's material to the "good" texture.
3. Render the scene with a dedicated camera into a `WebGLRenderTarget`.
4. `readRenderTargetPixels` into a 2D canvas.
5. Restore everything.

## 5. Tile preview camera — orthographic, fitted, with pole guard

Preview must look canonical regardless of where the tile is on the sphere
*and* match what the player sees on the globe. The fix:

- **OrthographicCamera, not PerspectiveCamera.** Perspective FOV is fixed and
  doesn't track the main view; orthographic gives a clean "head-on flat" view
  at all subdivision levels.
- **Fit the frustum to the tile's own vertex footprint.** Transform each
  vertex into camera space (`v.applyMatrix4(cam.matrixWorldInverse)`), take
  `max(|x|, |y|)` across vertices, multiply by ~1.06 for stroke padding, use
  as the half-extent.
- **Pole guard.** `lookAt` degenerates when the target direction is parallel
  to the up-vector — adjacent polar tiles render at random rotations:
  ```js
  if (Math.abs(fy) > 0.95) cam.up.set(0, 0, 1);
  else                     cam.up.set(0, 1, 0);
  ```

## 6. Clip path = projected vertices, not regular hexagon

A regular vertex-up hexagon mask:
- Doesn't match Voronoi cells (they're irregular by design).
- Doesn't match geodesic-dual hexes (slightly irregular near pentagons).
- Doesn't account for the tile's arbitrary rotation in camera space.

After rendering the preview, project each tile vertex to pixel coordinates
through the same camera and use those as the clip path:

```js
const clipPts = tile.verts.map(v => {
  const p = new THREE.Vector3(-v[0]*R, v[1]*R, v[2]*R).project(cam);
  return [(p.x + 1) / 2 * size, (1 - p.y) / 2 * size];
});
```

The `1 - p.y` is critical — must match the Y-flip in `readRenderTargetPixels`
output (rows come back bottom-to-top).

## 7. 2D screen overlay for carried/highlighted tiles

To draw a 2D HUD on top of a 3D tile:
- Project the tile's center AND every vertex with the **main** camera
  (`v.project(camera)`).
- Convert NDC → screen pixels: `x = (v.x+1)/2 * width`, `y = (-v.y+1)/2 * height`.
- Take the bounding box of all projected points → position+size for an
  HTML/canvas element.
- Skip drawing when `v.z > 1` (tile is behind the camera / on the far side
  of the sphere).

## 8. Voronoi-specific notes (since that's your next demo)

What rejected Voronoi for *Better Day*: tile-swap gameplay needs every tile
roughly the same shape, and Fibonacci-sphere Voronoi cells are too irregular
in size and outline. **None of that applies to non-swap demos.** For your
new demo, Voronoi is fine, and the rest of this doc still applies — just
substitute Voronoi cells for the geodesic-dual faces:

- Each cell's `center` is its generator point on the sphere.
- Each cell's `verts` are its boundary vertices, ordered (CCW from outside,
  CW from inside — pick one and stay consistent with your triangulation
  winding and `BackSide`).
- Triangulate each cell as a center-fan exactly like §3.
- The preview camera + clip-path code in §5–6 is shape-agnostic — it reads
  `tile.verts` and Just Works.

Watch out for:
- **Degenerate cells** at low generator counts (some cells get 3-4 vertices,
  making the orthographic frustum collapse on one axis). Floor `h` to a
  minimum (~5 px in target space) before applying.
- **Inconsistent winding** between cells if your Voronoi library isn't
  careful — backside culling will hide some tiles. Sort each cell's vertices
  by angle around the generator before using.
- **Generator distribution.** Pure random generators clump and leave gaps.
  Use Fibonacci/Golden-spiral sampling, or relax with a few Lloyd iterations.

## 9. Pixel readback gotcha

```js
renderer.readRenderTargetPixels(rt, 0, 0, size, size, px);
```

Returns rows **bottom-to-top**. Either flip while copying into the canvas
ImageData (Better Day does this), or compensate later — but be consistent.
Any vertex projection used as a clip path must use `(1 - ndc.y)/2` to match.

## 10. Things that failed — don't revisit

(From `memory.md`. Listed so you don't burn a day relearning.)

- Flat hex grid on equirectangular texture — pole distortion is unfixable.
- `cos(lat)` correction on hex grid — fixes spacing only, not shape.
- Cubemap with hex grids per face — seam artifacts at cube edges.
- Manual UV sampling for tile previews — see §4, just use render-to-texture.
- UV offset/swap for "scrambled" display — wrong region sampled at seams.
- Rendering tile A's content on tile B's mesh — UVs don't transfer; distorts.
- Custom shader-based hex grids — didn't compile on iPhone Safari.

## 11. Three.js / build constants worth remembering

- `R = 500` for sphere radius (with `near: 0.1, far: 1100`).
- `MeshBasicMaterial` (no lighting) is right for skyboxes — the panorama
  already has baked lighting; adding scene lights desaturates everything.
- WebGL `linewidth` doesn't work on most platforms. For bolder grid lines,
  draw the line geometry twice at slightly different radii.
- `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))` — capping
  at 2 saves a lot of fragment work on Retina iPhones with no visible loss.

---

**Reference implementation:** `panorama-viewer.html` in this repo.
**Key functions to copy/adapt:**
- `s2uv` (line 111) — equirectangular UV.
- `build()` (lines 147–201) — fan triangulation + seam fix.
- `renderTilePreview()` + `makePreview()` (lines 219–323) — preview pipeline.
