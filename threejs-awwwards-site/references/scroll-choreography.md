# Scroll choreography: binding scroll position to Three.js + GSAP

This is the core mechanism that makes this genre of site work: one continuous scroll timeline
drives everything (camera, 3D object transforms, DOM text reveals, section transitions) instead
of independent per-section animations.

## 1. Smooth scroll setup (Lenis) driving ScrollTrigger

```js
import Lenis from 'lenis';
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';

gsap.registerPlugin(ScrollTrigger);

const lenis = new Lenis({
  duration: 1.2,
  easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)), // exponential ease-out
  smoothWheel: true,
});

// Keep GSAP's ticker and Lenis in sync — critical, don't run two separate rAF loops.
lenis.on('scroll', ScrollTrigger.update);

gsap.ticker.add((time) => {
  lenis.raf(time * 1000);
});
gsap.ticker.lagSmoothing(0);
```

Common mistake: running Lenis's own internal `requestAnimationFrame` loop *and* a separate
`requestAnimationFrame` loop for Three.js rendering *and* GSAP's ticker, all unsynchronized. Drive
everything from a single ticker (GSAP's `gsap.ticker` is a good single source of truth) to avoid
visible judder between the 3D scene and DOM animation.

## 2. Driving a Three.js camera/object from scroll progress

Use ScrollTrigger to get a 0–1 progress value across a scroll distance, then apply it to Three.js
objects inside the render loop (don't mutate Three.js objects directly inside the ScrollTrigger
callback if you can avoid extra reflows — store a progress value and read it in the render loop):

```js
let scrollProgress = { value: 0 };

ScrollTrigger.create({
  trigger: '#scroll-container',
  start: 'top top',
  end: 'bottom bottom',
  scrub: 1, // smooths ScrollTrigger's response to scroll input; tune 0.5–2
  onUpdate: (self) => {
    scrollProgress.value = self.progress;
  },
});

function animate() {
  requestAnimationFrame(animate);

  // Example: dolly the camera along a path as the user scrolls
  camera.position.z = 10 - scrollProgress.value * 8;
  camera.position.y = Math.sin(scrollProgress.value * Math.PI) * 2;

  renderer.render(scene, camera);
}
animate();
```

For more complex camera paths, define a `THREE.CatmullRomCurve3` through a series of waypoints
and use `curve.getPointAt(scrollProgress.value)` rather than hand-writing per-axis easing —
much easier to art-direct and adjust waypoints later.

## 3. Chapter-based reveal pattern (matches the reference site's "Chapter 1 / Chapter 2..." structure)

```js
gsap.utils.toArray('.chapter').forEach((chapter, i) => {
  const tl = gsap.timeline({
    scrollTrigger: {
      trigger: chapter,
      start: 'top 80%',
      end: 'top 20%',
      scrub: 1,
      // markers: true, // enable during development, remove before shipping
    },
  });

  tl.from(chapter.querySelector('.chapter-heading'), { opacity: 0, y: 60, duration: 1 })
    .from(chapter.querySelector('.chapter-body'), { opacity: 0, y: 30, duration: 1 }, '-=0.6');
});
```

Pin a chapter in place while its 3D counterpart animates, if the design calls for a "scene stays
fixed while story text scrolls past" moment:

```js
ScrollTrigger.create({
  trigger: '.chapter-3d-pinned',
  start: 'top top',
  end: '+=200%', // stays pinned for 2x the viewport height of scroll distance
  pin: true,
  scrub: 1,
});
```

## 4. Section-to-section full-bleed transitions (video/image swap, matches reference site's video links)

A common pattern: hovering or scrolling into a nav item cross-fades a full-viewport background
video/image tied to that section, before navigating. Keep this snappy — cross-fade opacity over
~300-500ms, preload the video's poster frame or first frame so the swap doesn't flash blank.

## General cautions

- `scrub: true` (boolean) ties animation exactly to scroll with no smoothing lag; `scrub: <number>`
  (e.g. `scrub: 1`) adds a smoothing delay in seconds — most award-sites use a small numeric scrub
  value (0.5–1.5) for a slightly trailing, weighty feel rather than instant 1:1 tracking.
- Always call `ScrollTrigger.refresh()` after dynamically loaded content (images, fonts, GLTF
  models) finishes loading and changes document height, or trigger positions will be wrong.
- Kill/revert ScrollTrigger instances and GSAP timelines on component unmount or page
  navigation (especially relevant in a React/Vue/Svelte app) to avoid memory leaks and duplicate
  triggers on re-mount.
