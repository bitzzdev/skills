# Asset pipeline: models, textures, loading strategy

## Model format

- **glTF/GLB is the standard format** for Three.js scenes — prefer `.glb` (binary, single file)
  over `.gltf` (JSON + separate assets) for simpler deployment.
- Compress geometry with **Draco** or **Meshopt** compression for anything beyond a simple
  low-poly model — large uncompressed glTF files are a common cause of slow, janky first loads.
- Load with `GLTFLoader` + the matching decoder (`DRACOLoader` for Draco-compressed files):

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/draco/'); // path to the draco decoder wasm files, self-hosted or CDN

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

gltfLoader.load('/models/product.glb', (gltf) => {
  scene.add(gltf.scene);
});
```

## Texture strategy

- Compress/resize textures appropriately for how large they'll appear on screen — a 4K texture
  on an object that's a few hundred pixels tall is wasted bandwidth and memory.
- Use `.webp` or `.avif` for any 2D image assets (backgrounds, UI images) shown via HTML/CSS
  rather than as Three.js textures — matches the reference site's own use of `.webp` throughout.
- For Three.js textures specifically, consider KTX2/Basis compressed textures
  (`KTX2Loader`) for large scenes with many textures, to reduce GPU memory pressure — this
  matters more for complex scenes than a single hero object.
- Set `texture.colorSpace = THREE.SRGBColorSpace` on color textures (base color/albedo maps) so
  colors render correctly with the renderer's output color space; leave normal/roughness/metalness
  maps as linear (default).

## Loading sequence / preloader

Nearly every site in this genre gates the reveal behind a loading sequence rather than showing a
half-loaded page. Structure:

```js
const manager = new THREE.LoadingManager();

manager.onProgress = (url, loaded, total) => {
  const percent = Math.round((loaded / total) * 100);
  updatePreloaderUI(percent); // e.g. animate a progress bar / percentage counter
};

manager.onLoad = () => {
  // All tracked assets loaded — trigger the intro reveal animation (GSAP timeline),
  // then remove/hide the preloader overlay.
  playIntroSequence();
};

// Pass `manager` into loaders so they report progress to it:
const gltfLoader = new GLTFLoader(manager);
const textureLoader = new THREE.TextureLoader(manager);
```

Cautions:
- Don't block the preloader on assets that aren't actually needed for the first paint (e.g. video
  files for later sections) — only gate on what's needed for the hero, then lazy-load the rest as
  the user scrolls toward those sections.
- Give the preloader a minimum sensible display time or graceful exit animation even if loading
  is instant (cached assets) — an preloader that flashes for 40ms looks like a bug, not
  intentional pacing. A small artificial minimum (e.g. 400-600ms) plus a smooth transition out is
  common practice, but don't overdo it into an artificial multi-second delay that just frustrates
  repeat visitors.

## Video

For full-bleed background/section video (as in the reference site's `/assets/video/*.mp4` section
previews):
- Serve an appropriately compressed `.mp4` (H.264) with a `.webm` fallback if broad compatibility
  matters; keep file sizes down (short loops, reasonable bitrate) since these often autoplay.
- Use `muted`, `playsInline`, and `autoplay` attributes together for background video to satisfy
  mobile browser autoplay restrictions (`autoplay` alone without `muted`+`playsInline` is commonly
  blocked on mobile Safari/Chrome).
- Lazy-load/attach the video `src` only when the section is near viewport, rather than loading
  every section's video upfront.
