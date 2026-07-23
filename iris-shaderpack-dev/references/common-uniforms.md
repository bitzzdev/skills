# Common uniforms, attributes, and varyings

Starting-point reference. Availability of specific uniforms can vary by which program you're
in (not all uniforms are populated in every program) and by Iris version — cross-check
ShaderDoc (https://github.com/IrisShaders/ShaderDoc) for anything the requested effect
critically depends on, especially newer/less common uniforms.

## Matrices

- `gbufferModelView`, `gbufferModelViewInverse` — current model-view matrix and its inverse.
- `gbufferProjection`, `gbufferProjectionInverse` — current projection matrix and its inverse.
- `gbufferPreviousModelView`, `gbufferPreviousProjection` — previous frame's matrices (useful for motion vectors/TAA).
- `shadowModelView`, `shadowModelViewInverse` — shadow pass model-view matrix.
- `shadowProjection`, `shadowProjectionInverse` — shadow pass projection matrix.
- `modelViewMatrix`, `projectionMatrix`, `textureMatrix` — vertex-shader-only vanilla-style matrices in some contexts.

## Time / world state

- `worldTime` — the Minecraft day/night tick counter (0–24000).
- `frameTimeCounter` — seconds since shader pack load, useful for animation.
- `frameCounter` — integer frame count.
- `sunPosition`, `moonPosition` — direction vectors, view space.
- `shadowLightPosition` — whichever of sun/moon is currently the active shadow-casting light.
- `sunAngle`, `shadowAngle` — normalized sun angle values.
- `rainStrength` — 0.0–1.0, current rain intensity.
- `wetness` — smoothed rain strength, used for wet-surface effects.
- `isEyeInWater` — 0 = not in water, 1 = in water, 2 = in lava (check current version's exact semantics).

## Camera / player

- `cameraPosition` — world-space camera position, useful for world-space effects across chunk boundaries.
- `eyeAltitude` — camera's Y position.
- `isEyeInWater`, `isEyeInLava`... — environment-of-camera flags (naming/availability can vary; verify).
- `entityColor` — tint color for the current entity if applicable (hurt-flash red tint, etc), in gbuffers_entities.

## Textures / samplers (common ones)

- `gtexture` / `tex` (naming has changed historically — `texture` in modern format) — the block/entity texture atlas being drawn.
- `lightmap` — the vanilla light map texture (sky+block light baked in).
- `normals`, `specular` — PBR-style texture inputs if the pack supports resource-pack PBR (labPBR-style).
- `depthtex0`, `depthtex1`, `depthtex2` — depth buffers (main scene depth, translucents-excluded depth, etc).
- `colortex0` through `colortexN` — the general-purpose color buffers you can render to/read from across composite passes (max N depends on Iris version — verify current limit if the pack needs many buffers).
- `shadowtex0`, `shadowtex1` — shadow map depth buffers.
- `shadowcolor0`, `shadowcolor1` — shadow map color buffers (used for colored/translucent shadows, e.g. stained glass).
- `noisetex` — a built-in noise texture some packs use for dithering/randomization.

## Common vertex attributes / varyings

- `gl_Vertex`, `gl_MultiTexCoord0` (legacy-style) or modern `in vec3 vaPosition` / `in vec2 vaUV0` style attributes depending on GLSL version targeted.
- Custom varyings you declare yourself to pass data from vertex to fragment shader, e.g.
  `out vec2 texCoord;` in the vertex shader and matching `in vec2 texCoord;` in the fragment
  shader — always keep these consistent, mismatched varying declarations are a common source
  of silent shader errors.

## Caution

This list is a helpful starting point, not a guarantee of what's available in a specific
program in a specific Iris version. If a uniform doesn't seem to behave as expected, or the
user reports incorrect values, verify against ShaderDoc or ask the user to check
`latest.log` for shader compile errors (undeclared/mistyped uniforms often just silently
compile as 0/uninitialized in GLSL rather than erroring, which can be confusing to debug).
