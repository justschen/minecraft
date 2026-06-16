# Voxel Sandbox — Three.js Flow (Minecraft-style)

A browser-based, Minecraft-style voxel sandbox that runs with **no build step**, **offline**, and
previews in VS Code's **Integrated Browser**. This README documents an architecture/flow that works —
read it before building, or hand it to the agent as grounding context alongside `/minecraft-plan`.

> **Stack:** plain **Three.js** (vendored, ES modules via import map). No bundler, no framework.

---

## Why this flow

- **No build step** → `npx serve .` + open Simple Browser. Nothing to break on stage.
- **Offline-safe** → Three.js is vendored into `lib/`, loaded through an import map.
- **Agent-friendly** → Three.js voxel code is well-represented in training data, so agent mode
  generates predictable, working code.
- **One world, two tiers** → start with the *easy* renderer, upgrade only if you need scale.

## Project structure

```
minecraft-demo/
  index.html              # import map + canvas + UI overlay (crosshair, hotbar)
  lib/
    three.module.js       # vendored Three.js ES module build (offline)
    PointerLockControls.js# vendored from three/examples/jsm/controls
  src/
    main.js               # scene, camera, renderer, light, game loop
    blocks.js             # block registry: id → name, color/texture, solid
    world.js              # voxel storage + get/set + which chunks to remesh
    mesher.js             # builds renderable geometry from voxels
    player.js             # WASD + gravity + jump + AABB collision
    interaction.js        # raycast target, highlight box, break/place
    ui.js                 # hotbar + crosshair + selected block
  README.md
```

Keep it modular so the **parallel agent sessions** (textures, save/load, sound) touch different files.

---

## The two rendering tiers

### Tier 1 — `InstancedMesh` (fastest to working — start here)

One `BoxGeometry`, one `InstancedMesh` per block type (or one with per-instance color). Each visible
block is an *instance*. Place/remove = set or clear an instance matrix. **No custom mesher needed.**
Perfect for a demo-sized world (e.g. 64×16×64) and the quickest path to "walking around placing blocks."

```js
import * as THREE from 'three';

const geo = new THREE.BoxGeometry(1, 1, 1);
const mat = new THREE.MeshLambertMaterial(); // honors per-instance color
const mesh = new THREE.InstancedMesh(geo, mat, MAX_BLOCKS);
mesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);

const m = new THREE.Matrix4();
let i = 0;
for (const { x, y, z, color } of visibleBlocks()) {
  m.setPosition(x + 0.5, y + 0.5, z + 0.5);
  mesh.setMatrixAt(i, m);
  mesh.setColorAt(i, color);
  i++;
}
mesh.count = i;
mesh.instanceMatrix.needsUpdate = true;
scene.add(mesh);
```

**Trade-off:** renders all 6 faces of every block (including hidden interior faces). Totally fine for
a small world; wasteful for a big one → that's when you move to Tier 2.

### Tier 2 — Chunked geometry with hidden-face culling (scale/accuracy)

Split the world into chunks (16³ or 32³). For each chunk, build **one** `BufferGeometry` that emits a
quad **only where a solid voxel borders air** (hidden-face culling). On edit, **remesh just the
affected chunk** (and neighbors if the edit is on a chunk boundary). This is the "accurate Minecraft"
renderer and what `/minecraft-plan` asks the agent to confirm up front.

```
for each voxel v in chunk:
  if v is solid:
    for each of 6 directions d:
      if neighbor(v, d) is air:   # face is exposed
        push 4 verts + 2 tris + normal + uv for that face
upload positions/normals/uvs/colors as one BufferGeometry
```

> **Rule of thumb:** start Tier 1, ship Milestone 1, and only adopt Tier 2 if FPS drops or the world
> grows. Don't pre-optimize on stage.

---

## Core systems

### Blocks (`blocks.js`)
A registry keyed by integer id (`0 = air`). Each entry: `name`, per-face `color` (or texture/atlas
coords later), and `solid`. Keep it data-driven so adding "glowstone" is a one-line change live.

### World (`world.js`)
Voxel storage as a `Map<chunkKey, Uint8Array>` (or a flat `Uint8Array` for a fixed-size world).
`getBlock(x,y,z)` / `setBlock(x,y,z,id)`; `setBlock` marks the owning chunk **dirty** so the mesher
rebuilds only what changed.

### Player + controls (`player.js`)
- **Look:** `PointerLockControls` (vendored) — click to capture the mouse.
- **Move:** track velocity; WASD via `controls.moveForward/moveRight`; gravity on Y; Space to jump.
- **Collision:** PointerLockControls does **not** collide — do a simple **AABB sweep** against solid
  voxels, resolving one axis at a time (X, then Z, then Y) so you slide along walls and land on ground.

### Targeting + interaction (`interaction.js`)
Pick the block under the crosshair. Two options:
- **Easy:** `THREE.Raycaster` against the mesh → use `intersection.face.normal` (Tier 1 also gives
  `intersection.instanceId`).
- **Accurate:** a **voxel DDA raycast** (Amanatides & Woo) that steps cell-by-cell along the grid and
  returns the exact hit cell **and** the face normal — best for precise placement.

Then:
- **Left-click → break:** `setBlock(hit, 0)`.
- **Right-click → place:** `setBlock(hit + faceNormal, activeBlock)` (reject if it would intersect the
  player's AABB).
- Draw a thin **selection outline** (`THREE.BoxHelper` / wireframe) on the targeted cell + a center
  **crosshair** in the HTML overlay.

### Lighting / atmosphere (`main.js`)
`HemisphereLight` (sky/ground) + a `DirectionalLight` "sun"; set `scene.background` to sky-blue and
add light `THREE.Fog` for depth. Optional day/night = animate the sun + background color.

### Game loop (`main.js`)
```js
renderer.setAnimationLoop(() => {
  const dt = clock.getDelta();
  updatePlayer(dt);          // input + physics + collision
  updateTargetHighlight();   // raycast crosshair
  if (world.dirtyChunks.size) remeshDirtyChunks();
  renderer.render(scene, camera);
});
```

---

## Vendoring Three.js (offline) + import map

Copy two files into `lib/` (from the `three` package — `npm pack three` or download a release):

- `build/three.module.js` → `lib/three.module.js`
- `examples/jsm/controls/PointerLockControls.js` → `lib/PointerLockControls.js`

Then in `index.html`:

```html
<script type="importmap">
{ "imports": {
    "three": "./lib/three.module.js",
    "three/addons/controls/PointerLockControls.js": "./lib/PointerLockControls.js"
} }
</script>
<script type="module" src="./src/main.js"></script>
```

Now `import * as THREE from 'three'` works with **no bundler** and **no network**.

## Run it

```bash
cd minecraft-demo
npx --yes serve .          # serves at http://localhost:3000
# VS Code: Cmd/Ctrl+Shift+P → "Simple Browser: Show" → paste the URL
```

---

## Milestone map (matches `/minecraft-plan`)

| # | Milestone | You can see… | Tier |
|---|-----------|--------------|------|
| 1 | Scene + one chunk of blocks, look around | a voxel field, mouse-look | 1 |
| 2 | Flat/seeded terrain + block colors | grass/dirt/stone ground | 1 |
| 3 | Player physics: gravity, jump, collision | walking on terrain | 1 |
| 4 | Raycast targeting + highlight + crosshair | outlined block under crosshair | 1 |
| 5 | Break / place + hotbar (1–9, wheel) | building & mining | 1 |
| 6 | Atmosphere: sun, sky, fog | a lit, Minecraft-y world | 1 |
| — | **Parallelizable (separate agents):** textures · save/load (localStorage) · sounds · day/night | — | 1→2 |
| 7 | *(if needed)* chunked mesher + face culling | same world, 60 FPS at scale | 2 |

## Live-demo increments (safe, high-impact)
- **Glowstone:** add a block type that emits light (`PointLight` at placed cells, or emissive material).
- **Day/night on `N`:** lerp sun intensity + `scene.background` over a few seconds.

## Performance notes
- Tier 1: cap world size; reuse one `Matrix4`/`Color`; `DynamicDrawUsage` on the instance buffer.
- Tier 2: remesh **only dirty chunks**; share geometry attributes; let frustum culling drop offscreen chunks.
- Target **60 FPS** on a laptop; if it dips during the demo, shrink the world radius — don't debug live.
