# Three.js scene setup

A standard, sane bootstrap for a full-viewport WebGL hero scene. Adapt object/material choices
to the actual design direction — this is the plumbing, not the art direction.

## Basic renderer / scene / camera

```js
import * as THREE from 'three';

const canvas = document.querySelector('#webgl-canvas');

const scene = new THREE.Scene();

const camera = new THREE.PerspectiveCamera(
  35, // fov — narrower (25-40) reads as more "product photography", wider (50-75) more immersive
  window.innerWidth / window.innerHeight,
  0.1,
  100
);
camera.position.set(0, 0, 8);

const renderer = new THREE.WebGLRenderer({
  canvas,
  antialias: true,
  alpha: true, // transparent background if compositing over HTML/CSS behind it
});

// Cap pixel ratio — rendering at full devicePixelRatio (often 2-3 on modern displays) with
// post-processing is a common, easily avoidable performance killer.
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping; // common choice for a cinematic, filmic look
renderer.toneMappingExposure = 1.0;
```

## Resize handling

```js
function onResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}
window.addEventListener('resize', onResize);
```

Debounce this if resize-triggered layout is expensive (recomputing complex geometry, etc); a
simple `setSize` call is cheap enough to run un-debounced in most cases.

## Render loop

Prefer a single `requestAnimationFrame` loop that also drives GSAP/Lenis updates from the same
tick (see `scroll-choreography.md`), rather than letting Three.js, GSAP, and Lenis each run
independent loops:

```js
function animate() {
  requestAnimationFrame(animate);
  // update scroll-driven values, controls, mixers, etc. here
  renderer.render(scene, camera);
}
animate();
```

If using `OrbitControls` or similar for a draggable scene (less common in this genre, since
camera movement is usually scroll-driven rather than user-dragged, but sometimes combined):

```js
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
// remember to call controls.update() each frame if damping is enabled
```

## Lighting basics for a product-style hero (matches the reference site's clean, soft-lit look)

```js
const keyLight = new THREE.DirectionalLight(0xffffff, 2);
keyLight.position.set(3, 5, 4);
scene.add(keyLight);

const fillLight = new THREE.AmbientLight(0xffffff, 0.4);
scene.add(fillLight);

// Environment map for realistic reflections on product-like materials (metal, glass, coated surfaces)
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
new RGBELoader().load('/textures/studio.hdr', (hdri) => {
  hdri.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = hdri; // lights PBR materials without needing a visible skybox
});
```

## Post-processing (used sparingly and deliberately, not by default)

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  0.4,  // strength — keep subtle; strong bloom reads as a demo reel, not a product site
  0.6,  // radius
  0.85  // threshold
));
// Then call composer.render() instead of renderer.render(scene, camera) in the loop.
```

Post-processing multiplies render cost — only add passes the design actually calls for, and
always check the performance guidance in `performance-and-fallbacks.md` before shipping heavy
effects.

## Module source

Import Three.js addons (`OrbitControls`, `RGBELoader`, post-processing passes, `GLTFLoader`,
`DRACOLoader`, etc.) from `three/addons/...` when using a bundler with the `three` npm package —
this is the current convention (replacing the older `three/examples/jsm/...` path pattern still
seen in older tutorials/blog posts, which may still work but isn't the current recommended
import path). Confirm the current recommended import convention against threejs.org's own docs
or examples if something doesn't resolve, since Three.js's own internal module layout has shifted
across versions.
