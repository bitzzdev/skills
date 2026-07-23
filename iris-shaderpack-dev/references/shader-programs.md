# Common shader program names (Iris/OptiFine format)

This is a starting-point reference, not exhaustive or guaranteed current for the newest
Iris/Minecraft versions — cross-check https://github.com/IrisShaders/ShaderDoc for anything
version-sensitive or for programs not listed here.

Each program typically has a `.vsh` (vertex) and `.fsh` (fragment) file, e.g.
`gbuffers_terrain.vsh` + `gbuffers_terrain.fsh`.

## Gbuffers programs (render the actual scene geometry)

- `gbuffers_basic` — fallback/basic geometry, lines, selection outlines.
- `gbuffers_textured` — simple textured geometry without lighting (e.g. some particles).
- `gbuffers_textured_lit` — textured geometry with lighting applied.
- `gbuffers_terrain` — solid terrain blocks (most world geometry).
- `gbuffers_terrain_cutout` / `gbuffers_terrain_cutout_mip` — cutout-alpha blocks (leaves, etc).
- `gbuffers_damagedblock` — block break/crack overlay.
- `gbuffers_water` — translucent terrain: water, ice, stained glass.
- `gbuffers_skybasic` — sky, stars, void plane.
- `gbuffers_skytextured` — sun/moon textured quads.
- `gbuffers_clouds` — vanilla clouds.
- `gbuffers_weather` — rain/snow particles.
- `gbuffers_entities` — mobs/players (non-hand).
- `gbuffers_entities_glowing` (naming varies by version) — glowing entity outline effect (e.g. Glowing status effect).
- `gbuffers_hand` — the player's held item/hand, first person.
- `gbuffers_hand_water` — held item when translucent (rare).
- `gbuffers_block` — block entities (e.g. some tile entities rendered as block-like models).
- `gbuffers_beaconbeam` — beacon beam effect.
- `gbuffers_item` — dropped items / GUI-rendered items in some contexts (verify per version).
- `gbuffers_armor_glint` / `gbuffers_glint` — enchantment glint overlay.
- `gbuffers_spidereyes` — spider/enderman eye glow (naming varies).

## Shadow programs (shadow map pass, optional)

- `shadow` — the shadow-casting geometry pass.
- `shadowcomp` — a composite-style pass operating on the shadow buffer, if supported.

## Composite / deferred / post-processing programs

- `prepare` (and `prepare1`, `prepare2`, ...) — optional early passes before deferred lighting, for precomputing buffers.
- `deferred` (and `deferred1`, `deferred2`, ...) — deferred lighting pass(es), operating on the gbuffer data.
- `composite` (and `composite1`, `composite2`, ... up to a version-dependent max) — post-processing passes chained in order; common home for fog, bloom, god rays, reflections.
- `final` — the last pass before the image is presented; commonly used for tone mapping, vignette, final color grading.

Numbered variants (`composite1`, `composite2`, etc.) run in sequence, each able to read the
previous pass's output — check ShaderDoc or the target Iris version's changelog for the
current maximum number of composite passes supported, since this has been raised over time.

## Notes

- Not all programs need to be implemented — Iris/OptiFine fall back to a default/vanilla-like
  behavior for any program you don't override.
- Exact availability and behavior of some programs (e.g. newer entity-related gbuffers
  programs, or shadow-related composite passes) has changed across Iris versions — if the
  user names a specific recent version or an effect that depends on one of the less common
  programs above, verify against ShaderDoc rather than assuming this list is complete/current.
