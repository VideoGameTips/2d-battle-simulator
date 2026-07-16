# Iron Tide — Graphics Design Brief

A workspace for **redesigning the game's visuals**. Everything you see in Iron Tide is
generated procedurally in code — there are **zero art assets** (no textures, models, or
images). So "editing the graphics" means editing JavaScript in **`../index.html`**.

This document maps every visual system to its exact location, records the current values,
and flags concrete redesign targets so a design pass can jump straight in.

---

## Hard constraints (please keep these)

- **One file, no dependencies.** The whole game is a single self-contained `index.html`:
  vanilla JS + Three.js **r128** (loaded from CDN) + Canvas-generated textures. No build,
  no assets, no server. Don't add binary assets or npm packages.
- **Verify before done:** extract the `<script>` and run `node --check`:
  ```
  python3 -c "import re;open('/tmp/c.js','w').write(re.search(r'<script>(.*)</script>',open('index.html').read(),re.S).group(1))" && node --check /tmp/c.js
  ```
- **Geometry helpers:** `mkBox(w,h,d,mat)` and `mkCyl(rt,rb,h,mat,seg)` build meshes; most
  models are boxes/cylinders welded into a `THREE.Group`. Canvas textures come from
  `softTex()` / `surfaceMap()` (radial gradients + value-noise — still zero-asset).
- **Balance transforms:** builders nest many `group.add()` / `save/restore`-style transforms.
  Keep them balanced.

## Current art direction

A deliberately **readable, semi-stylized "Roblox-ish" military look** — matte, flat-ish,
not photoreal. ACES filmic tone mapping at low exposure keeps it legible rather than dark.
The honest weaknesses (per the owner: *"you codebots are no artist"*): hard-edged un-beveled
primitives, flat shading with **no ambient occlusion**, **no bloom**, a too-calm sea, and
no material micro-detail. Those are the redesign opportunities below.

---

## Visual systems → code map

Line numbers are approximate (the file changes); search the function name to be safe.

### 1. Materials & colour  — `index.html` ~479–499, 1828
- `surfaceMap(kind)` (~479): canvas value-noise textures. Kinds: `metal`, `paint`, `concrete`, `ground`.
- `surfaceMaterial(color, kind, roughness, metalness)` (~495): the base `MeshStandardMaterial` factory.
- `SILVER = c => surfaceMaterial(c, 'metal', 0.62, 0.38)` — metal parts (barrels, tracks, hardware).
- `PAINT  = c => surfaceMaterial(c, 'paint', 0.90, 0.02)` — matte painted bodies (tank/plane/ship hulls).
- `aircraftSkin(col)` (~1828): matte painted airframe (`paint`, ~0.66 rough, ~0.08 metal).
- **Redesign targets:** add real **normal maps** (panel lines, rivets, weld seams); **weathering**
  (rust streaks, salt stains, scorch near exhausts) via a second overlay texture or vertex colours;
  **per-faction palettes** (blue vs. red already exist for flags — extend to hull tints).

### 2. Lighting & tone  — `index.html` ~508–518
- `renderer.toneMapping = ACESFilmicToneMapping`, `toneMappingExposure = 0.98`.
- `renderer.shadowMap.enabled = false` — **shadows are OFF** (they were re-rendering ~20k meshes
  twice; huge cost). This is a big part of the "flat" feel.
- `sun = DirectionalLight(0xfff0d8, 1.25)`; `hemi = HemisphereLight(0xc8d6e2, 0x344252, 0.78)`.
- Sun/hemi intensity + colour are re-driven every frame by time-of-day in `updateWeather` (~825).
- **Redesign targets:** re-enable **cheap contact shadows** (a blob/decal shadow under each hull, or a
  single low-res shadow map just for big units); add a subtle **rim/fill light**; stronger **dusk/dawn
  colour grading**; a faint **specular sun-glint** streak on the water.

### 3. Post-processing  — `index.html` ~537–617  (currently mostly OFF)
- `EffectComposer` + `RenderPass` + a custom **`AOShader`** (screen-space AO from depth) + `UnrealBloomPass`.
- `gfxQuality = 0` by default → **AO disabled** (its depth-derived normals showed noise over the flat ocean);
  **bloom is opt-in** (`UnrealBloomPass(res, strength 0.22, radius 0.35, threshold 1.12)` — threshold set high
  so matte hulls don't glow, only very bright pixels).
- Toggle in-game with **`I`** (`cycleGfxQuality`). `loop()` renders through the composer when `gfxQuality>0`.
- **Redesign targets:** this is the **single highest-leverage** area. Get **SSAO working without ocean
  noise** (mask by depth/normal, or restrict to units), then a **tasteful bloom** (fire, muzzle flashes,
  the sun disc, tracers). Known snag: an earlier AO attempt hit `GL_INVALID_OPERATION` on the depth-texture
  format — verify framebuffer completeness (`gl.checkFramebufferStatus`) and depth-texture type.

### 4. Sky & atmosphere  — `index.html` ~649–800
- `initGraphics()` (~673): `sunSprite` (glow billboard), `starPts` (night star dome), `cloudSprites`
  (drifting billboards), `rainGroup`.
- `updateGraphics(dt,t)` (~760): drives sun position/opacity, star fade, cloud drift & tint, foam rings,
  ship wakes, damage smoke.
- `scene.background` / `scene.fog` colour + fog distance are re-tinted per weather in `updateWeather` (~838).
- **Redesign targets:** layered/volumetric-looking clouds; **god rays** from the sun sprite; a richer sky
  **gradient** (horizon→zenith) instead of a flat colour; **aurora / moon** on clear nights; heat-haze near fires.

### 5. Water  — `index.html` ~636, 659
- `makeWaterNormals()` (~659): smoothed value-noise → tangent-space **normal map** (scrolled each frame).
- `animateWater(t)` (~636): displaces a `PlaneGeometry(18000,18000,48,48)` with two sine waves, scaled by
  `weather.sea`; scrolls the normal map.
- **Redesign targets:** biggest surface on screen and the calmest-looking. Add **Gerstner waves** (sharper
  crests), **whitecaps / foam** at crests and around hulls & islands, a **sun-glint** highlight, and
  cheap **screen-space reflections** or a fake fresnel horizon band. Sea state should read very differently
  calm vs. storm.

### 6. Ships  — `index.html` ~1258 (`buildShip`), ~3824 (`buildAIShipModel`)
- `buildShip(def, enemy)`: hull, deck, superstructure, funnels, bridge, radar, rails, flags — all from
  `mkBox`/`mkCyl`. `SHIPS` catalog (~230) defines `len/beam/deckY/hp/kind/mounts` per hull (30+ ships incl.
  legendary Yamato, Bismarck, Enterprise, Akagi, etc.).
- `buildAIShipModel` wraps `buildShip` and bolts on the team marker + boss guns.
- **Redesign targets:** **bevel the hard edges** (chamfered boxes or `RoundedBoxGeometry`-style helpers);
  **panel lines / deck planking** via textures; more **silhouette variety** between hull classes; portholes,
  catwalks, better bridges. Carriers (`kind:'carrier'`) get a flat flight deck — add deck markings/elevators.

### 7. Tanks & land units  — `index.html` ~1604 (`buildLand`), ~1779 (`buildTurret`)
- `buildLand(kind,team,pos,type)`: tanks (hull, tracks, road wheels, glacis, turret, mantlet, barrel,
  cupola, MG, fenders, exhaust, antenna, stowage), plus coastal guns, AA, artillery, silos, structures.
- `TANKS` catalog (~1183). Tank bodies use `PAINT(T.col)`; barrels/tracks use metal.
- **Redesign targets:** bevels; track **tread texture**; muddy/dusty lower hull; tank commander figures.

### 8. Aircraft & drones  — `index.html` ~1961 (`buildPlane`)
- `buildPlane(def)` branches on `def.shape`: `prop | jet | attack | wing | heli | heavyprop | quad | vtol`.
  Helpers: `aircraftFuselage`, `aircraftWingSet`, `aircraftCanopy`, `aircraftPropeller`, `aircraftFin`,
  `aircraftUnifiedAirframe`. `PLANES` catalog (~285) incl. WWII props, jets, bombers, and **8 UAV types**
  (quad, VTOL w/ sensor ball, nuclear kamikaze).
- **Redesign targets:** panel lines & national insignia decals (canvas textures); afterburner/exhaust glow
  (ties into bloom); smoother fuselage lofts.

### 9. Harbours & islands  — `index.html` ~1517 (`buildHarbor`), ~1572 (islands/structures)
- `buildHarbor` + `addIslandBuildingGraphicDetails`: docks, cranes, radar, silos, airstrips, helipads, walls,
  fortress turrets. Islands get procedural terrain, rocks, and trees (thousands via instancing).
- **Redesign targets:** shoreline foam (partly exists via `_foamRings`), beach/sand gradient, more building
  variety, lit windows at night.

### 10. Combat FX  — `index.html` ~5018–5300, ~847
- `boom(pos,scale)` (~5053): layered explosion — white flash, fireball, smoke column, embers, debris, light.
- `fireParticle`/`burnFx` (~5185/5209) + `fireFx` (~5273): the shared **fire** system (flame tongues, oily
  smoke, embers, flicker glow) used by burning ships/tanks/planes/buildings.
- `muzzleFlash` (~5018), `splashAt` (~5026, shell/water geysers), `lightningBolt` (~847, storm strikes),
  `crippleParts` (~5217, battle-damage: buckle/scorch/shed parts). Shell tracers are in `spawnShell`.
- **Redesign targets:** smoke as **soft sprite sheets** rather than blobs; **heat-shimmer** post-effect near
  big fires; longer-lived **oil slicks / debris fields** on sinkings; better water-impact rings.

---

## Highest-leverage redesign order (impact ÷ effort)

1. **Post-processing: SSAO + tasteful bloom** — fixes "flat" and "no glow" at once, globally. (§3)
2. **Water** — biggest surface on screen; Gerstner chop + whitecaps + sun-glint. (§5)
3. **Contact shadows** — grounds every object; even fake blob shadows help enormously. (§2)
4. **Material micro-detail** — normal maps + weathering so surfaces aren't flat plastic. (§1)
5. **Bevels on geometry** — softens the hard-edged "prototype" silhouette. (§6–8; hardest, do last.)

## The wider repo

Iron Tide is `index.html`. The repo also holds ~20 other self-contained single-file games
(`clash.html`, `mech-battles.html`, `world-conquest2.html`, `stick-fighter.html`, `tron.html`,
`laststand.html`, `plane.html`, …). Each is the same "one HTML file, procedural graphics, no
assets" shape, so the same techniques above apply if the redesign expands to them.
