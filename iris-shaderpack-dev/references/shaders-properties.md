# shaders.properties syntax reference

`shaders.properties` sits in `shaders/shaders.properties` and configures pack metadata,
user-facing options, and per-program overrides. It's a Java-style `.properties` file
(`key=value`, one per line, `#` for comments).

## Defining a toggleable option (checkbox)

```properties
# Declares a boolean-like shader option the user can toggle in the shader GUI.
# The actual #define must exist in your GLSL source; this just exposes a toggle for it.
screen=<screen_name>
```

The more common pattern is: declare a `#define SOME_FEATURE` in a shared `.glsl` include
file, then expose it as an option:

```properties
option.SOME_FEATURE=Enable Some Feature
```

Sliders (numeric ranges) look like:

```properties
option.FOG_DENSITY=Fog Density
option.FOG_DENSITY.default=1.0
option.FOG_DENSITY.min=0.0
option.FOG_DENSITY.max=5.0
```

(Exact slider syntax has some version-to-version variance — verify against ShaderDoc /
Iris-Example-Shaderpack if sliders aren't behaving as expected, rather than guessing at key names.)

## Organizing options into screens/pages

```properties
screen=<MAIN_SCREEN_NAME>
screen.<MAIN_SCREEN_NAME>=OPTION_A OPTION_B OPTION_C
sliders=FOG_DENSITY
```

## Profiles (preset combinations of options)

```properties
profile.low=OPTION_A:false OPTION_B:false
profile.high=OPTION_A:true OPTION_B:true
```

## Per-program overrides

Blending mode for a specific program:

```properties
blend.composite=SRC_ALPHA ONE_MINUS_SRC_ALPHA
```

Render targets / buffer format overrides:

```properties
# Request a specific format/size for a colortex buffer used across composite passes
colortex0.format=RGBA16F
```

Custom render targets for extra composite passes, buffer clearing behavior
(`clear.colortex1=false` to persist a buffer across frames, useful for temporal effects like
simple TAA or auto-exposure accumulation), and render stage inclusion/exclusion
(`program.gbuffers_water=composite1` style tags in some versions) also exist — always check
ShaderDoc for the exact current key names before inventing one, since properties keys are
easy to get subtly wrong (they fail silently — a mistyped key is just ignored rather than
causing an error).

## General cautions

- Unrecognized/mistyped keys are typically ignored silently rather than erroring — if an
  option doesn't seem to take effect, double check exact spelling/casing against
  ShaderDoc/Iris-Example-Shaderpack rather than assuming the GLSL side is at fault.
- Keep `shaders.properties` in sync with actual `#define`s in your GLSL — an option exposed
  in properties but not referenced anywhere in shader code does nothing (harmless but
  confusing to the user), and a `#define` used in code but not exposed in properties can't be
  toggled from the in-game GUI.
