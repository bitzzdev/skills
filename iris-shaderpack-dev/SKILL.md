---
name: iris-shaderpack-dev
description: Use this skill whenever the user asks to create, build, edit, or debug a Minecraft shaderpack for Iris (or OptiFine-compatible shaders, since Iris targets OptiFine's shader format). Trigger on phrases like "make a shader", "iris shaderpack", "write a fragment shader for minecraft", "add fog/water/clouds/god rays to my shader", "fix my shaders.properties", or any request to touch .vsh/.fsh/.glsl files, shaders.properties, or a shaderpacks/<name>/shaders folder. ALSO use this skill (do not skip it) whenever the user mentions a specific new Minecraft or Iris version — Iris's supported shader features, gbuffers program names, and uniform availability have changed across versions (e.g. the 1.17+ core shaders rewrite, 1.19+ additions, 1.21+ multiloader/NeoForge support), and the agent's memorized knowledge of these may be stale. This skill enforces looking up current Iris/ShaderDoc documentation before writing shader code for anything version-sensitive, rather than guessing GLSL uniforms, program names, or shaders.properties keys from memory.
---

# Iris Shaderpack Development

## What this covers

Writing and editing Minecraft shaderpacks compatible with **Iris** (the open-source, actively
maintained shaders mod) and, by extension, mostly compatible with **OptiFine**'s shader format,
since Iris intentionally implements the same shaderpack format OptiFine popularized. A
shaderpack is a folder of GLSL shader programs plus a `shaders.properties` config file — not
a Java mod. No Gradle/Loom/Java is involved here (that's the separate `minecraft-fabric-mod`
skill, for actual Fabric mods written in Java).

## Core principle: don't guess GLSL uniforms, program names, or version-gated features from memory

Shaderpack development is unusually easy to get subtly wrong because:
- Program/file names follow specific conventions (`gbuffers_terrain.fsh`, `composite1.fsh`,
  `shadow.vsh`, etc.) that must match exactly or Iris silently won't use them.
- Available uniforms, and which ones are populated in which program, have grown/changed
  across Iris versions and Minecraft versions.
- `shaders.properties` option/key syntax (sliders, profiles, `program.<name>` overrides,
  conditional defines) is easy to misremember or invent plausible-but-wrong keys for.
- OptiFine's own documentation is unlicensed and must never be copied/reproduced verbatim —
  only IrisShaders' own ShaderDoc project may be quoted from, and even then sparingly per
  normal copyright rules (paraphrase, don't reproduce large blocks).

**Therefore: before writing shader code touching a feature the agent isn't fully certain about
(a specific uniform, a newer gbuffers program, a shaders.properties feature, or anything tied to
a Minecraft/Iris version the user names), search first.** Prefer these sources, in order:

1. `https://github.com/IrisShaders/ShaderDoc` — the authoritative, Iris-maintained shader format docs (getting-started guides, reference pages on uniforms, programs, textures, buffers).
2. `https://github.com/IrisShaders/docs` — Iris's own user/dev docs site source.
3. `https://github.com/IrisShaders/Iris/releases` — changelogs, useful for confirming what a specific Iris version added/changed.
4. `https://github.com/IrisShaders/Iris-Example-Shaderpack` — a real minimal working shaderpack, useful for confirming correct file structure/naming when unsure.
5. Community shaderpack source (e.g. well-known open-source packs on GitHub/Modrinth) as secondary confirmation, only for structure/convention checks, never to copy code wholesale — respect the licenses of any shaderpack repo you look at, and never copy a substantial code block from someone else's pack; use it only to confirm your own from-scratch code is structured correctly.

Never reproduce OptiFine's own documentation text even paraphrased-heavily; it's proprietary. Use ShaderDoc as the primary reference instead.

## Step 1 — Clarify the target and scope

If not already clear, figure out:
- Target Iris version / Minecraft version (matters for available uniforms & program names — search if a recent/new version is named).
- What the user wants: a full shaderpack from scratch, or a specific effect added to an existing pack (fog, waving foliage, water reflections, shadows, bloom, god rays, colored lighting, etc).
- Whether they want OptiFine compatibility too, or an Iris-exclusive pack (Iris-exclusive packs can use newer features and should show an error screen on OptiFine — see the Iris-Example-Shaderpack pattern for this).

## Step 2 — Folder/file structure

A shaderpack lives at `shaderpacks/<PackName>/` and needs at minimum:

```
<PackName>/
├── shaders/
│   ├── shaders.properties        (optional but almost always present)
│   ├── gbuffers_basic.vsh / .fsh (and other gbuffers_* programs you override)
│   ├── composite.fsh / .vsh      (optional post-processing passes)
│   ├── final.fsh / .vsh          (optional final pass)
│   └── shadow.vsh / .fsh         (optional, only if implementing shadows)
└── pack.png                       (optional, shown in shader selection screen)
```

Key rules:
- You only need to write `.vsh`/`.fsh` files for the **programs you want to override** —
  Iris/OptiFine fall back to defaults for anything you don't provide. Don't scaffold every
  possible gbuffers program if the user only wants one effect.
- File names are meaningful and must match Iris's expected program names exactly (case
  sensitive). If unsure of the exact name for a program relevant to the user's request
  (e.g. "which program handles the sky" or "which program handles item entities"), check
  ShaderDoc rather than guessing — there are many similarly-named programs
  (`gbuffers_entities`, `gbuffers_hand`, `gbuffers_block`, `gbuffers_weather`, etc.) and
  picking the wrong one means the effect silently doesn't apply where expected.
- Read `references/shader-programs.md` for a working reference list of common program names
  and what they're for — but re-verify against ShaderDoc if the user's MC/Iris version is
  recent or if the requested effect touches a program not covered there.

## Step 3 — Writing GLSL

- Target GLSL 150 (OpenGL 3.2) at minimum for modern Minecraft/Iris; Iris/Sodium allow
  higher versions if the user's hardware supports it. Old OptiFine-only packs used GLSL 120 —
  ask if backward compatibility with very old Minecraft/OptiFine is actually wanted before
  writing legacy-style code.
- Read `references/common-uniforms.md` for frequently-used uniforms (`gbufferModelView`,
  `gbufferProjection`, `worldTime`, `sunPosition`, `cameraPosition`, `frameTimeCounter`, etc)
  and common attributes/varyings passed between vertex and fragment shaders.
- Read `references/shaders-properties.md` for `shaders.properties` syntax: defining sliders,
  toggleable options, profiles, and program-specific overrides like blending or
  `render.gbuffers_water=...`.
- Structure new effects as an isolated composite pass where reasonable (e.g. a
  `composite.fsh` doing a post-process fog/bloom pass over the rendered scene) rather than
  cramming everything into gbuffers programs — easier for the user to debug and toggle.

## Step 4 — Common effects quick-pointers

Read `references/common-effects.md` for a starting-point sketch of frequently requested
effects (fog, waving grass, water reflection/refraction, simple bloom, colored torch light,
underwater tint, basic shadows). These are illustrative skeletons meant to be adapted, not
copy-pasted verbatim as a finished product — always adjust to the user's actual pack
structure and confirm any version-sensitive uniform against ShaderDoc first.

## Step 5 — Testing guidance

Since this environment cannot run Minecraft, always tell the user:
- Where to place the shaderpack folder (`.minecraft/shaderpacks/`).
- To enable **debug mode** in the shader options (Iris has an in-game debug toggle) if
  something doesn't load — it surfaces GLSL compile errors instead of failing silently.
- What log output to check (`latest.log` in the `.minecraft/logs` folder) if a shader fails
  to compile, and to paste back any GLSL compiler error so it can be fixed precisely rather
  than guessed at.

## When editing an existing shaderpack

If the user has an existing pack and wants a change/fix, read the actual files they've
provided/uploaded rather than assuming their structure — real packs vary a lot in how they
organize composite passes and buffers. Match their existing conventions (naming, indentation,
existing uniform usage) rather than imposing a different style.
