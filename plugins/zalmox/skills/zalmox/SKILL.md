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
   - Motion for a model: `animate`.
   - Explosions, fire, smoke, magic, electricity, thrusters as a flipbook: `vfx-flipbook`, or `vfx-layered` for a
     full explosion, fire or impact with lit smoke, sparks and a light (`duration` sets how long a one-shot plays;
     an impact defaults to 0.5 s). Pure smoke simulations: `vfx-smoke`. VFX jobs also save their JSON sidecar
     (layers, grid, fps) loose as a `sidecar` output, so there's no need to unpack the Unity package to read it.
   - A tiling material: `pbr-texture`. Sky: `skybox`. Landscape: `terrain`. Concept art or icons: `image`.
4. **Wait and save.** Jobs take one to ten minutes. `wait_for_job` with a `download_dir` inside the user's project
   (e.g. `Assets/Zalmox/` in a Unity project) saves the files; if it says the job is still running, call it again.
   CLI: `--wait --out <dir>`.
5. **Report what you saved**: the files and what each is for. The `.unitypackage` imports into Unity (Assets →
   Import Package) and builds a ready prefab. `.glb` / `.fbx` are the model, and flipbook PNGs are sprite sheets
   (columns × rows are in the job's outputs).
6. **Don't retry blindly.** A failed job returns its credits. Read the error, change what it points at (prompt,
   image, option), and tell the user before trying again.

## Prompts that work

- 3D: one object, what it is, its material and style ("a weathered wooden treasure chest with iron bands, stylized").
  No scenes or several objects.
- Images for `image-to-3d`: the whole object, plain background, three-quarter view.
- VFX: the effect's colours and character ("violet arcane burst with gold sparks"), and choose `effect` (explosion,
  fire, smoke, vortex, magic, electric, lightning, thruster). `vortex` is a tornado, dust devil or whirlpool spinning in
  place; `electric` an arc between two points; `lightning` a strike from the sky that flashes and fades. Loops
  (`loop: on`) suit fire, smoke, vortices, electric arcs and thrusters. `colour: neutral` gives a white/grey effect
  to tint in the game. Results vary a lot by seed: `variations: 2` or `4` (priced per variation) makes several at
  once with a contact sheet (`contact_sheet` output) to pick from.
- `vfx-smoke`: try `preview: on` first (6×6 frames, 128 px, a coarse simulation, a fraction of the time and credits,
  draft only), then run the chosen seed full size. `color` only tints the material: for several colours make one
  smoke and recolour it in Unity; for different smoke change the seed.
- Every job type that makes files takes a `name` (e.g. "storm wall" → `storm_wall_1a2b3c`): set one when making
  several assets of one kind, so their files and prefabs can be told apart.
