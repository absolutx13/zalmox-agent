---
name: zalmox
description: Generate game-ready assets with Zalmox (zalmox.io) - 3D models from text or images, retextured or animated models, VFX flipbooks, PBR textures, skyboxes, terrains and images, with Unity packages. Use when the user wants game art, a 3D model, a Unity-ready asset, a particle/VFX sprite sheet, a seamless texture or a skybox made for them.
---

# Zalmox

Zalmox makes game-ready assets on the user's account. Use the `zalmox` MCP tools if they're available; otherwise the
`zalmox` CLI (`npx zalmox …`, add `--json` for machine-readable output). If neither is signed in, ask the user to
run `npx zalmox login` once.

## Rules

1. **Money first.** Cloud jobs spend the user's credits; `my-gpu` jobs are free but need their paired machine to be
   on (to set one up, follow https://zalmox.io/runner/SETUP.md with the user, or send them to https://zalmox.io/my-gpu). Before a cloud job, get the price (`estimate_cost`, or `zalmox types <type>`) and tell the user; for anything
   over about 50 credits, or several jobs, ask before starting. Pass `max_credits` / `--max-credits` as a guard.
2. **Draft first.** Start on the draft tier; offer final (more candidates, judged, costs more) once the user likes
   the direction.
3. **Pick the right job type** (`list_job_types` / `zalmox types` has every option):
   - A 3D model from a description: `text-to-3d`. From a picture: `image-to-3d` (`image_path` / `--image`).
   - A new look for an existing model: `retexture` (its `sourceModel` is the model asset's id).
   - Motion for a model: `animate` (see "Animating a model" below for its `motions` list).
   - Explosions, fire, smoke, magic, electricity, thrusters as a flipbook: `vfx-flipbook`, or `vfx-layered` for a
     full explosion, fire or impact with lit smoke, sparks and a light (`duration` sets how long a one-shot plays;
     an impact defaults to 0.5 s). Pure smoke simulations: `vfx-smoke`. VFX jobs also save their JSON sidecar
     (layers, grid, fps) loose as a `sidecar` output, so there's no need to unpack the Unity package to read it.
   - A tiling material: `pbr-texture`. Sky: `skybox`. Landscape: `terrain`. Concept art or icons: `image`.
4. **Wait and save.** Jobs take one to ten minutes. `wait_for_job` with a `download_dir` inside the user's project
   (e.g. `Assets/Zalmox/` in a Unity project) saves the files; if it says the job is still running, call it again.
   CLI: `--wait --out <dir>`. A queued cloud job whose stage reads "Starting a cloud GPU (about N min)" is waiting
   for a GPU to boot (about 7 minutes when none is running): tell the user and keep waiting; it isn't stuck.
5. **Report what you saved**: the files and what each is for. The `.unitypackage` imports into Unity (Assets →
   Import Package) and builds a ready prefab. `.glb` / `.fbx` are the model, and flipbook PNGs are sprite sheets
   (columns × rows are in the job's outputs). A model whose picture showed lights (a glowing eye, a light strip) also
   comes with `<name>_emissive.png`, already the emission map of the GLB and of the Unity material: use it for glow
   rather than deriving one from the albedo (`emissive: off` skips it; no file means no lights were found, or that
   the picture's view couldn't be told from its opposite: the model output's `emissive.reason` says which).
   Animating the model keeps the map (`emissive.carried` in the animated model's metadata).
6. **Don't retry blindly.** A failed job returns its credits. Read the error, change what it points at (prompt,
   image, option), and tell the user before trying again.

## Prompts that work

- 3D: one object, what it is, its material and style ("a weathered wooden treasure chest with iron bands, stylized").
  No scenes or several objects.
- Images for `image-to-3d`: the whole object, plain background, three-quarter view.
- Triangle budgets for 3D: `targetFaceCount` (e.g. 25000) or `lowPoly`. When the budget is 150,000 or fewer (a tenth
  or less of the generated surface), or `lowPoly` is set, Zalmox also bakes a normal map from the detailed surface
  (`<name>_normal.png`, in the Unity package too), so plating and small detail survive the cut; it adds about half a
  minute. The model output's metadata records it as `normalBake` (`reason`, `from`: the detailed mesh's triangles). If
  the bake couldn't be made, its `checks` has a `normalMap` entry saying why: tell the user, as the detail is missing.
- Two-sided objects (drones, vehicles, crates, furniture) can come out with uneven sides: rotors at different
  heights, one arm fused into the hull. `symmetry: auto` (with `straighten: on`) mirrors the better half onto the
  other when the object is clearly symmetric, and leaves it alone otherwise; the model output's metadata has
  `symmetry` (score 0–1, `applied`, `kept`, or the `reason` it wasn't mirrored; from 0.65 up it mirrors unless the
  picture disagrees, and from 0.55 when the model's sides face an axis, which `straighten: on` gives, and the picture agrees). `symmetry: x` forces a mirror
  across the left-right (X) plane. Leave it off for characters holding something, plants and rocks.
- Backs of characters, capes and packs: `backView: generate` draws the object from behind first and models the shape
  from both views. The drawing is checked against the picture (its outline must be the picture's mirror image, in the
  same colours; on Final the vision model checks it isn't the front again), redrawn once if it fails, and left out
  if that fails as well.
  Read the model output's `backView` metadata: `used`, `score` (0–1), `rejected` (each with `problems`). When `used`
  is false the model was made from the front alone and `checks` has a `backView` entry: tell the user. It works on
  pictures seen straight from the front; three-quarter views from above rarely pass, so leave it off for those. The
  check can't see a part invented inside the outline: look at the `back_view.png` output when the back matters.
  `backView: force` uses the first drawing whatever the check says. Use it only after looking: when a `generate` job's
  refused drawing (`back_view_rejected_1.png`) shows the same object from behind with nothing added, run again with the
  same `seed` and `force`; the output's `checks` then has a `backViewPose` entry reminding you to look at the back.
- Textures (`pbr-texture`): describe the surface and its colours ("dry cracked desert earth with pebbles, warm
  ochre"). Don't write "seamless", "tileable" or "no symmetry": the tiling is done for you, and naming symmetry tends
  to paint it. `material` sets the kind of surface only; colours come from the description. `size` is 1024 (default),
  2048 or 4096 pixels per side (8 and 12 credits instead of 5). To turn the user's own picture into a material, pass
  it as the image (`image_path` / `--image`) with a prompt that says what it shows: its centre square is made to
  tile (a narrow band along its edges is repainted so that they meet; the rest stays as it is and where it is,
  `seam.at` is `edges`), and the normal, height, roughness and AO maps are derived from it. A picture that already tiles (a painted
  tile, a texture from a library) is recognised and kept exactly as it is: `tileable` is `auto` by default (its
  edges are measured), `yes` keeps it without measuring, `no` repaints its seam anyway; the metadata's `picture`
  entry and `seam.kept` say what happened. The measuring can be wrong on fine, even surfaces (dust, sand): if a kept
  picture shows a faint seam where it repeats, make it again with `tileable: no`; if a picture that did tile was
  repainted (`seam.kept` false), with `tileable: yes`. Pictures whose opposite edges look alike
  (the same kind of surface left and right, top and bottom) join best; where they differ, the repainted band shows
  as a strip of its own along the tile's edges. The albedo output's metadata has
  `symmetry` (`flagged`: the layout is mirrored and reads as a kaleidoscope when tiled, even after its one re-roll)
  and `seam` (`flagged`: a seam still shows through the middle of the tile). If either is flagged, tell the user and
  try another seed. `reliefStrength` sets how steep the normal map is (its steepest tenth tilts about 30, 45 or 60
  degrees; `normalTilt` in the metadata): medium suits most surfaces, also rock under a strong light. A rock face or
  other deep relief often comes out lit, with shadow bands in the albedo (`brightness.dark` under about 0.85 in the
  metadata): `delight: on` evens them out and keeps the relief in the normal map; leave it off for surfaces with
  dark and light parts of their own. Shine is not read from the description: when it says glossy, wet, glassy, icy
  or polished, set `roughness` (`matte`, `satin`, `glossy`, `wet`: a mean smoothness of 10, 28, 45, 60 percent) or
  `smoothness` (0 to 100, the mean itself; it overrides the level). Left on `auto` the map follows `material`, and
  ground or stone come out nearly fully rough whatever the picture shows. The picture places the shine round that
  mean: dark parts shine, pale and grainy parts stay dull. For a ground seen at a low angle (a game's terrain) keep
  it low, since the whole plain mirrors the sky: a dark ground loses its colour above about 12 to 18, a bright one
  takes 28 to 45, so `satin` is the shiny level there and `glossy` or `wet` are for bright surfaces or ones seen
  from above. The metadata's `roughness` entry (`level`, which is `custom` when a figure was given; `smoothness`, `mean`, `smooth`,
  `rough`) says what came out.
- VFX: the effect's colours and character ("violet arcane burst with gold sparks"), and choose `effect` (explosion,
  fire, smoke, vortex, magic, electric, lightning, thruster). `vortex` is a tornado, dust devil or whirlpool spinning in
  place; `electric` an arc between two points; `lightning` a strike from the sky that flashes and fades. Loops
  (`loop: on`) suit fire, smoke, vortices, electric arcs and thrusters. `colour: neutral` gives a white/grey effect
  to tint in the game. Results vary a lot by seed: `variations: 2` or `4` (priced per variation) makes several at
  once with a contact sheet (`contact_sheet` output) to pick from. Name the fire and what flies from it, never what
  bursts or what it stands on: "a cannon shell bursting on the ground" draws the shell, and "on rocky ground" a
  rock, as a hole or a notch at the base of the fireball in every frame. A one-shot never ends on a bright frame: when
  the clip was cut while still burning, its last third is faded to nothing (`fadedOut`: how many frames, in the
  flipbook's metadata and on a layered effect's generated layer; absent: it ended by itself).
- `vfx-smoke`: try `preview: on` first (6×6 frames, 128 px, a coarse simulation, a fraction of the time and credits,
  draft only), then run the chosen seed full size. `color` only tints the material: for several colours make one
  smoke and recolour it in Unity; for different smoke change the seed.
- Skyboxes (`skybox`): the `.hdr` output's metadata (and `<name>_sky.json`) says where to light the scene from:
  `sunDirection`, with `sunFrom` saying how it was found: `disc` (a painted sun, also `sunUV`), `glow` (no disc, the
  brightest part of the sky: `sunBearing` and `sunElevation`) or `default` (nothing stood out: overcast, night,
  space; a soft light from high up, not a measurement).
- Sound effects (`sfx`): say only what should be heard. The model has no negative prompt and naming a sound tends
  to produce it, so "no explosion, no music" parts are left out of what it is given (the clip's metadata lists them
  as `unsaid`). A one-shot is cut to its sound and is never longer than `length`; a loop (`loop: on`) is the steady
  part of the take, so it can be a little shorter than `length`: read `seconds`. A loop keeps its own rise and fall
  (gusts, chimes, cracks); pass `evenness: bed` for a hum, a drone or an engine, which should sit at one level
  (`sound.swingDb` says how far a loop's level moves: a bed a dB or two, an ambience six or more). Every loop comes
  at about -20 LUFS, so loops of different jobs sit level with each other: drips or clanks over a quiet ground get
  there with their sharpest peaks held down (`sound.limitedDb`, at most 8 dB; absent: none). Pick takes from each
  `.ogg` output's metadata without listening: `sound` has `lufs`, `peakDb`, `rmsDb`, `soundSeconds`, the `rumble`
  share that was removed (under 28 Hz) and the `bass` / `mid` / `high` shares of its energy (split at 150 Hz and
  8 kHz); `checks` lists what is wrong with a take (`bass`: nearly all under 150 Hz, silent on small speakers;
  `silence`; `rumble`; `quiet`; `short`: a loop under two thirds of the `length` asked, because the take faded
  away). A flagged take was already made a second time and the better one kept (`rerolled`),
  so a take that is still flagged needs different words: prompts naming something low ("deep rumble", "low hum")
  often give bass only; add what is heard above it ("with a hissing, crackling top"). The one-shots of one job come
  at one loudness and none is louder than -16 LUFS (louder takes are lowered: `gainDb`), so any of them can be
  played at random and a sustained tone sits with the hits beside it. The words set the sound's shape as well as its
  kind: the model places events on a fixed grid, so "a hard crack and a short tail" at `length` 1 comes as two
  separate sounds in every take, and a longer `length` changes the character, not only the padding. `sound.onsets` lists the
  seconds at which a second sound starts inside a one-shot (absent: none). A subject that implies an ending draws
  it whatever the words say ("a shell falling" ends in an impact): pick a take without onsets, cut before the first
  one, or describe only the sound itself ("a long whistling whine falling in pitch").
- Every job type that makes files takes a `name` (e.g. "storm wall" → `storm_wall_1a2b3c`): set one when making
  several assets of one kind, so their files and prefabs can be told apart.

## Animating a model

`animate` (2 credits, CPU) bakes motion into one of the user's models: `sourceModel` is the model asset's id (a 3D
job's GLB), `motions` a JSON list. `zalmox types animate` (or `list_job_types`, its `guide`) has the full format.

- Points are in the source GLB's own coordinates: glTF, Y up, metres. Read the model's size and where its parts
  are from the GLB before writing any.
- Whole-model presets, each once: `{"type":"bob"}`, `spin`, `hover`, `pulse`, `sway`, `recoil`. How far they move
  scales with the model.
- Up to 8 moving parts. A painted part is the faces whose centre lies in any of its `spheres` (`[x, y, z, radius]`):
  - `{"type":"wheel","name","spheres","pivot":[x,y,z],"axis":[x,y,z],"speed":1}` spins; speed is turns a second
    (0.25-4).
  - `{"type":"hinge","name","spheres","pivot","axis","angle":100}` opens by angle degrees (5-180), then closes.
  - `{"type":"slide","name","spheres","direction":[x,y,z],"distance":0.2}` moves out, in metres, then back.
  - Optional `"cut":[px,py,pz,nx,ny,nz]` and `"bounds":[min xyz, max xyz]` split the mesh cleanly along a plane.
- A propeller, fan or rotor disc needs no spheres: `{"type":"rotor","name","hub":[x,y,z]}` finds it from one point on
  its hub (its top or centre), cuts it free of its arm, caps the cut, puts the pivot on its centre and spins it
  (`speed`, 0.25-8 turns a second, default 3). Optional: `"axis"` (default up, turned to a tilted rotor's own plane;
  give `[0,-1,0]` for one hanging under its arm), `"radius"` and `"depth"` (metres: how far out, and how far under
  the hub along the axis, the rotor reaches).
  - Hub only works on clean rotors: a disc or duct, a rotor on a shaft above its arm, blades on a round motor.
  - Generated props are often irregular or fused to a neighbour's blades. Then the job fails asking for the radius, or
    the radius it reports is clearly not the rotor's: give `radius` and `depth`, read off the GLB. With both, the
    rotor is everything inside that cylinder, which always works.
  - A wheel can take `"hub"` (with its `"axis"`) instead of spheres and pivot, and is found the same way.
- Each motion is one clip, one after another on one timeline, with an Animator state in the Unity prefab: `Hover`
  and the other presets; `<Name>Spin` for a wheel or rotor, `<Name>Open`/`<Name>Close` for a hinge or slide. The
  name's words are run together, each starting with a capital, its own capitals kept ("front fan" becomes
  `FrontFanSpin`, "RotorFL" `RotorFLSpin`); parts need different names.
- To play clips together (hover while the rotors spin), layer them in the game. The clips share one take, so every
  animated node has keys in every clip, at rest where the clip doesn't move it: keep in each clip only the curves of
  the nodes it moves, listed as `nodes` in the report's `clips` (`<model>_Motion` for a preset, `<model>_<Name>_Pivot`
  for a part); pick curves by node, not by how little they move. A preset's clip also has keys on the collider hull
  `UCX_<model>_00`, the opposite of the model's, so the collider holds still while the model hovers: drop them to have
  it follow. The prefab's own controller plays one whole-model clip and one part clip at a time.
- A bad list is refused before any credits are spent, with the reason (e.g. `fan: the pivot is missing`).
- The model is cleaned and placed again (bottom centre on the origin, scaled to its height), so the points you gave
  move. The job's `<name>_motions.json` output (a `sidecar`, also in the model output's metadata) says how:
  `placement` (`scale`, `offset`: placed = scale x source + offset) and, per part, `pivot` and `axis` in the placed
  model, `sourcePivot` as given, and for a rotor `selection` (island, above the arm, above the motor, cylinder),
  `radius`, `cuts` and `centred`. Read it after every animate job and tell the user if a rotor's radius looks wrong.

A quadcopter about 0.5 m across, hovering, its four rotors spinning; the back two are fused to their arms, so they
get a radius and a depth:

```json
[{"type":"hover"},
 {"type":"rotor","name":"rotor front left","hub":[-0.2,0.12,0.2]},
 {"type":"rotor","name":"rotor front right","hub":[0.2,0.12,0.2]},
 {"type":"rotor","name":"rotor back left","hub":[-0.2,0.12,-0.2],"radius":0.09,"depth":0.02},
 {"type":"rotor","name":"rotor back right","hub":[0.2,0.12,-0.2],"radius":0.09,"depth":0.02}]
```

CLI: `zalmox create animate --sourceModel <asset id> --motions "$(cat drone-motions.json)" --wait --out ./assets`.
