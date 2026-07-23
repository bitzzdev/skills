# Common effect skeletons

These are illustrative starting points to adapt to the user's actual pack — not drop-in
finished code. Always adjust uniform/varying names to match the rest of their pack, and
verify any uniform you're unsure about against ShaderDoc before relying on it.

## 1. Simple distance fog (composite.fsh)

```glsl
#version 150

uniform sampler2D colortex0;   // scene color from gbuffers passes
uniform sampler2D depthtex0;   // scene depth
uniform mat4 gbufferProjectionInverse;
uniform mat4 gbufferModelViewInverse;

uniform float far;             // render distance, in blocks
uniform vec3 fogColor;

in vec2 texCoord;
out vec4 fragColor;

// Reconstructs view-space position from depth buffer + screen UV.
vec3 getViewPos(vec2 uv, float depth) {
    vec4 clip = vec4(uv * 2.0 - 1.0, depth * 2.0 - 1.0, 1.0);
    vec4 view = gbufferProjectionInverse * clip;
    return view.xyz / view.w;
}

void main() {
    vec3 color = texture(colortex0, texCoord).rgb;
    float depth = texture(depthtex0, texCoord).r;

    vec3 viewPos = getViewPos(texCoord, depth);
    float dist = length(viewPos);

    float fogFactor = clamp(dist / far, 0.0, 1.0);
    fogFactor = fogFactor * fogFactor; // quadratic falloff, adjust to taste

    color = mix(color, fogColor, fogFactor);
    fragColor = vec4(color, 1.0);
}
```

Corresponding minimal `composite.vsh` just needs to pass through a fullscreen-triangle UV as
`texCoord` — check `Iris-Example-Shaderpack` for the exact fullscreen-pass vertex shader
convention if unsure.

## 2. Waving foliage (gbuffers_terrain_cutout.vsh sketch)

```glsl
#version 150

uniform float frameTimeCounter;

// Iris/OptiFine expose a "mc_Entity" style attribute or block-id-based tagging in some
// versions to identify plant blocks for waving; the exact mechanism (attribute name,
// mod_control-style constants) varies — verify against ShaderDoc before relying on a
// specific attribute name here.

void main() {
    vec4 position = gl_Vertex; // or the modern `in vec3 vaPosition` equivalent

    // Sketch only — real waving logic keys off vertex Y-position within the block and a
    // per-block "is this a wavable plant" flag, then offsets X/Z with a sine wave:
    // float wave = sin(frameTimeCounter * speed + position.x + position.z) * amplitude;
    // position.x += wave * topVertexMask;

    gl_Position = gl_ModelViewProjectionMatrix * position; // or modern equivalent matrices
}
```

## 3. Simple screen-space water tint/refraction hint (composite.fsh sketch)

```glsl
// Sketch: sample colortex0 with a small UV offset derived from a noise texture and time,
// only where depth indicates translucent water was drawn (check isEyeInWater / a
// water-specific mask buffer, depending on how the pack tracks "is this pixel water").
vec2 distortedUV = texCoord + sin(texCoord.y * 40.0 + frameTimeCounter) * 0.002;
vec3 refracted = texture(colortex0, distortedUV).rgb;
```

## 4. Basic bloom (two-pass sketch: composite1 = bright-pass + blur, composite2 = combine)

```glsl
// composite1.fsh: extract bright areas
vec3 color = texture(colortex0, texCoord).rgb;
float brightness = dot(color, vec3(0.2126, 0.7152, 0.0722)); // luminance
vec3 bloomSource = color * step(0.8, brightness); // crude threshold, tune as needed
fragColor = vec4(bloomSource, 1.0); // written to a colortex buffer for blurring/combining
```

```glsl
// composite2.fsh: combine blurred bloom buffer with the original scene color
vec3 sceneColor = texture(colortex0, texCoord).rgb;
vec3 bloom = texture(colortex1, texCoord).rgb; // assuming colortex1 holds the blurred bloom
fragColor = vec4(sceneColor + bloom * bloomStrength, 1.0);
```

A real bloom implementation needs an actual blur pass (gaussian or box blur across several
composite passes/downsamples) — this sketch only shows the extract/combine structure.

## 5. Simple shadow sampling (used in a gbuffers or composite program, after implementing shadow.vsh/.fsh)

```glsl
uniform sampler2D shadowtex0;
uniform mat4 shadowModelView;
uniform mat4 shadowProjection;

// worldPos: world-space position of the fragment being shaded, relative to camera
float getShadow(vec3 worldPos) {
    vec4 shadowClip = shadowProjection * shadowModelView * vec4(worldPos, 1.0);
    vec3 shadowNDC = shadowClip.xyz / shadowClip.w;
    vec3 shadowScreen = shadowNDC * 0.5 + 0.5;

    float shadowDepth = texture(shadowtex0, shadowScreen.xy).r;
    return shadowScreen.z - 0.001 > shadowDepth ? 0.0 : 1.0; // 0.001 = bias, tune to avoid acne
}
```

Real packs usually add PCF (percentage-closer filtering) by sampling a small kernel around
`shadowScreen.xy` rather than a single tap, to soften shadow edges.

## Reminder

Every skeleton above uses illustrative uniform/attribute names and simplified logic. Before
finalizing code for the user, confirm any uniform, attribute, or program name you're not
fully certain of against ShaderDoc — especially if the user's Iris/Minecraft version is
recent or if they mention something not covered in these reference files.
