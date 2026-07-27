# Performance and fallback checklist

Treat this as a required pre-ship checklist, not optional polish, for any real (non-throwaway
prototype) build in this genre.

## WebGL availability / fallback

- Detect WebGL support and fail gracefully rather than showing a blank canvas or console errors:

```js
function isWebGLAvailable() {
  try {
    const canvas = document.createElement('canvas');
    return !!(window.WebGLRenderingContext &&
      (canvas.getContext('webgl') || canvas.getContext('experimental-webgl')));
  } catch (e) {
    return false;
  }
}

if (!isWebGLAvailable()) {
  // Show a static image/poster-frame fallback for the hero instead of the 3D scene,
  // and skip Three.js initialization entirely.
  showStaticFallback();
} else {
  initThreeScene();
}
```

- Also handle `webglcontextlost` (can happen on GPU driver crashes, tab backgrounding on mobile,
  too many WebGL contexts open) by listening for the event and either attempting
  `renderer.forceContextRestore()` patterns or falling back gracefully rather than leaving a
  frozen black canvas.

## Reduced motion

```js
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (prefersReducedMotion) {
  // Disable scroll-scrubbed camera moves, parallax, and cursor-follow effects.
  // Still show all content — just via simple fades/cuts instead of continuous scroll-driven motion.
  gsap.globalTimeline.timeScale(1); // or restructure ScrollTriggers to use simple `toggleActions`
  // instead of `scrub`, and skip initializing cursor-follow / magnetic-hover listeners.
}
```

Reduced motion means "reduce or remove non-essential motion," not "remove content or
functionality" — the story/information must still be fully accessible, just delivered with less
or no continuous animation.

## Mobile / low-end device considerations

- Cap `renderer.setPixelRatio()` (see `threejs-scene-setup.md`) — don't render at full native
  pixel ratio on high-DPI phones, it's a large, often invisible-to-the-eye performance cost.
- Consider disabling expensive post-processing passes (bloom, DOF, heavy particle counts) below
  a device/viewport-width threshold, or reducing particle/instance counts for mobile.
- Consider whether the 3D hero is even appropriate on small viewports — many award-sites swap to
  a simpler static or video-based hero on mobile rather than running the full WebGL scene, given
  GPU/battery/thermal constraints on phones. Ask the user if unclear whether full mobile parity
  matters for their use case, since it changes scope meaningfully.
- Test/consider frame budget: a scene with heavy post-processing that runs at 30fps on a mid-range
  phone will feel worse than a simpler scene at a smooth 60fps — favor cutting effects over
  shipping something that stutters.

## Resource cleanup

- Dispose Three.js geometries, materials, and textures (`.dispose()`) when a scene or route is
  torn down (SPA navigation, component unmount, or Vite HMR during development) — undisposed
  WebGL resources leak GPU memory and can eventually crash the tab.
- If using a framework (React/Vue/Svelte), tie Three.js scene setup/teardown to the component
  lifecycle explicitly rather than assuming a single global scene that never needs cleanup.

## Accessibility beyond motion

- Ensure DOM content (headings, body text, nav links) exists as real accessible HTML, not baked
  into canvas/WebGL or images-of-text — screen readers and SEO both need real text content, even
  on a heavily animated site. The 3D/WebGL layer should be a visual enhancement layered behind or
  alongside real DOM content, not a replacement for it.
- Keep focus order and keyboard navigation working through scroll-jacked/pinned sections —
  test tabbing through the page; pinned ScrollTrigger sections can sometimes trap or skip focus
  if not handled carefully.
- Maintain reasonable color contrast for text overlaid on video/3D backgrounds — a common failure
  mode in this genre is text that looks fine over one frame of a background video but becomes
  unreadable over lighter/busier frames later in the loop.

## Quick smoke-test before calling it done

1. Load with browser dev tools' network throttled (e.g. "Fast 3G") — does the preloader/loading
   sequence behave sensibly, or does it appear broken/stuck?
2. Toggle `prefers-reduced-motion` in dev tools and confirm content is still fully navigable.
3. Resize to a mobile viewport and confirm nothing is broken (even if the experience
   intentionally simplifies).
4. Check the browser console for WebGL/shader warnings or errors.
5. Tab through the page with keyboard only and confirm focus is visible and in a sensible order.
