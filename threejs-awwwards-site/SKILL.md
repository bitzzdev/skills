---
name: threejs-awwwards-site
description: Use when the user wants to build, redesign, or prototype a visually striking, awwwards-style website featuring Three.js or WebGL 3D scenes, scroll-driven animation, smooth scrolling, and cinematic page transitions, the genre of sites like agency showcases, product launch pages, or portfolios with a 3D hero, chapter-based scroll storytelling, and heavy motion polish built with Three.js, GSAP ScrollTrigger, Lenis smooth scroll, WebGL shaders, and post-processing. Trigger on phrases like "make a cool three.js website", "site like an awwwards winner", "3D hero section", "scroll-driven website with WebGL", or a request referencing a specific award-site-style URL as inspiration. Covers the technical architecture: smooth scroll, scene setup, scroll-to-3D binding, performance, loading sequences. Pair with the frontend-design skill for the visual identity pass, since this is about making the motion and 3D layer work, not picking colors and type.
---

# Three.js / Awwwards-style Site Builder

## What this genre actually is

Sites like the reference example (otsuka-air.jp) belong to a recognizable genre: full-viewport
WebGL/Three.js hero scenes, buttery smooth-scroll (usually virtual/inertial scroll, not native),
scroll-position-driven 3D and text animation ("chapters" that reveal as you scroll), full-bleed
video/image transitions between sections, custom cursor and hover micro-interactions, and a
strong loading sequence (preloader) before anything is shown. The defining trait isn't any
single library — it's the **choreography**: scroll position drives a single continuous timeline
that controls camera movement, 3D object transforms, text reveals, and section transitions
together, rather than each section animating independently.

This skill is about building that choreography correctly and performantly. For the actual visual
identity (palette, type, layout personality, copywriting) use the `frontend-design` skill's
process first or alongside this one — a technically perfect scroll rig with generic/templated
visual design will still look generic. Read `frontend-design`'s SKILL.md for that pass; this
skill assumes a design direction exists or is being developed in parallel.

## Step 1 — Clarify scope before building

Figure out, asking if genuinely ambiguous rather than assuming:
- **New site vs. redesign of existing content.** If redesigning, what content/copy/assets
  already exist? Read them before inventing new copy.
- **What's the 3D content actually of?** A literal product/object (bottle, device, character),
  an abstract generative scene (particles, fluid, noise-driven geometry), or a stylized
  environment? This determines whether Three.js primitives + a loaded GLTF model are needed, or
  a fully shader-driven abstract scene (which leans on the `iris-shaderpack-dev` skill's GLSL
  knowledge in spirit, though this is WebGL/GLSL for browsers, not Minecraft).
- **Delivery target**: a single-file HTML/JS artifact/prototype for quick iteration, or a real
  project (Vite/npm project with proper build tooling) meant to be deployed? Default to a real
  Vite-based project scaffold for anything beyond a single quick prototype — production
  award-sites are never single HTML files, and asset loading (GLTF, textures, video) benefits
  from real bundling.
- **Performance target**: does it need to work well on mobile/low-end GPUs? Award-site demos are
  often desktop-first and gracefully degrade elsewhere — confirm expectations rather than
  assuming full parity is required, since that changes how aggressive to be with post-processing.

## Step 2 — Core stack

Standard, current stack for this genre (verify exact latest versions via npm/web search if the
user cares about pinning specific versions — don't assume version numbers from memory for fast
moving libraries like Three.js, which ships frequent releases):

- **Three.js** — the 3D/WebGL layer itself.
- **GSAP + ScrollTrigger** — the industry-standard choice for binding scroll position to
  animation timelines; also good for non-3D DOM animation (text reveals, transitions).
- **Lenis** (by Studio Freight / Darkroom Engineering) — the current standard smooth-scroll
  library that replaced older approaches; provides inertial virtual scrolling and integrates
  cleanly with ScrollTrigger via `lenis.on('scroll', ScrollTrigger.update)`.
- **A bundler** — Vite is the standard default for a from-scratch Three.js project; use it
  unless the user has an existing framework (Next.js, Nuxt, SvelteKit, Astro — the fetched
  reference site is actually built on Astro, worth matching if the user wants that stack).
- **React Three Fiber (`@react-three/fiber`) + drei**, optionally, if the user's stack is React
  and they want declarative Three.js — otherwise use vanilla Three.js, which is more common and
  more flexible for this specific award-site genre.

Read `references/scroll-choreography.md` for the core pattern binding scroll position to a
Three.js scene and to GSAP timelines — this is the single most important technical piece to get
right, and the piece most likely to be built sloppily (e.g. driving the 3D scene straight off
native `window.scrollY` instead of through Lenis, causing jank).

## Step 3 — Scene and asset setup

Read `references/threejs-scene-setup.md` for a standard renderer/camera/scene bootstrap,
resize handling, and a note on `requestAnimationFrame` vs Three.js's built-in render loop helpers.

Read `references/asset-pipeline.md` for guidance on 3D model formats (glTF/GLB is the standard,
compressed with Draco or Meshopt for large scenes), texture compression, and lazy/progressive
loading — asset loading strategy directly determines whether the preloader experience feels
intentional or just slow.

## Step 4 — Motion & interaction layer

Read `references/motion-patterns.md` for concrete recipes: scroll-triggered chapter reveals,
camera dolly/pan tied to scroll progress, parallax layering, custom cursor, magnetic
hover-buttons, and page-load intro sequences (the "preloader then reveal" pattern nearly every
site in this genre uses).

## Step 5 — Performance and accessibility floor

Read `references/performance-and-fallbacks.md` before considering the build finished. Sites in
this genre are notorious for skipping: a graceful fallback for browsers/devices without solid
WebGL support, `prefers-reduced-motion` handling, disposing Three.js resources on navigation/HMR,
and keeping frame budget sane (avoid rendering at full resolution + heavy post-processing on
every frame without a pixel-ratio cap). Treat these as required, not optional polish, the same
way `frontend-design` treats a responsive/accessible floor as required for visual design.

## Step 6 — Deliverable

For a full project, scaffold real files (Vite project structure: `index.html`, `src/main.js` or
similar, `src/scene/`, `package.json`) rather than a single giant script — this is a "long,
multi-file build" case per the general file-creation guidance, so plan the file structure first,
then build incrementally, and mention the user can run `npm install && npm run dev` to preview
locally. For a quick prototype/exploration, a self-contained HTML artifact importing Three.js
from a CDN (e.g. `https://cdn.jsdelivr.net/npm/three@<version>/build/three.module.js` — confirm
the actual latest version rather than guessing) is fine, and can be shown live if the environment
supports rendering HTML artifacts.

## When redesigning based on a specific reference site

If the user points to a specific site (like the otsuka-air.jp example) as inspiration:
- Fetch it and look at its actual structure/copy/section flow before designing — don't assume
  what it contains from the name alone.
- Identify the underlying *pattern* (chapter-based scroll narrative, video-per-section reveal,
  interview/testimonial carousel, etc.) rather than copying its literal content, layout, brand
  colors, or copy verbatim — the goal is "a site in this genre for the user's own subject," not a
  clone. Apply `frontend-design`'s principle of grounding the design in the user's actual
  subject matter, not the reference site's.
- Never reproduce a reference site's copy text, and don't recreate distinctive proprietary visual
  assets (their exact photography, logos, or brand marks) — build original content structured the
  same way.
