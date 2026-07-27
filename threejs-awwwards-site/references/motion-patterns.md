# Motion & interaction pattern recipes

Concrete, adaptable recipes for the specific micro-interactions that give this genre its polish.
Tune timings/easings to the actual design direction rather than using these values verbatim.

## 1. Intro/load reveal sequence

```js
function playIntroSequence() {
  const tl = gsap.timeline();
  tl.to('.preloader', { opacity: 0, duration: 0.6, onComplete: () => {
      document.querySelector('.preloader').style.display = 'none';
    }})
    .from('.hero-heading .word', { y: '100%', stagger: 0.08, duration: 1, ease: 'power4.out' }, '-=0.3')
    .from('.hero-subtext', { opacity: 0, y: 20, duration: 0.8 }, '-=0.6')
    .from(canvas, { opacity: 0, duration: 1.2 }, '-=1');
}
```

Splitting heading text into per-word or per-character spans for staggered reveals is common —
either hand-wrap words in `<span>` tags at build time, or use a splitting utility. Keep this
simple: over-fragmenting text (per-character on long paragraphs) hurts readability and
accessibility (screen readers) if not handled carefully — prefer per-word or per-line splitting
for body text, per-character only for short display headlines where the effect is the point.

## 2. Custom cursor

```js
const cursor = document.querySelector('.custom-cursor');
let mouseX = 0, mouseY = 0, cursorX = 0, cursorY = 0;

window.addEventListener('mousemove', (e) => {
  mouseX = e.clientX;
  mouseY = e.clientY;
});

function animateCursor() {
  // Lerp toward the real mouse position for a trailing, weighty feel
  cursorX += (mouseX - cursorX) * 0.15;
  cursorY += (mouseY - cursorY) * 0.15;
  cursor.style.transform = `translate(${cursorX}px, ${cursorY}px)`;
  requestAnimationFrame(animateCursor);
}
animateCursor();
```

Grow/change the cursor on hover over interactive elements:

```js
document.querySelectorAll('a, button, [data-cursor-hover]').forEach((el) => {
  el.addEventListener('mouseenter', () => cursor.classList.add('cursor--hover'));
  el.addEventListener('mouseleave', () => cursor.classList.remove('cursor--hover'));
});
```

Hide the custom cursor entirely on touch devices (`@media (hover: hover) and (pointer: fine)`)
rather than showing a broken/stuck custom cursor on mobile.

## 3. Magnetic button hover

```js
document.querySelectorAll('[data-magnetic]').forEach((btn) => {
  btn.addEventListener('mousemove', (e) => {
    const rect = btn.getBoundingClientRect();
    const relX = e.clientX - rect.left - rect.width / 2;
    const relY = e.clientY - rect.top - rect.height / 2;
    gsap.to(btn, { x: relX * 0.3, y: relY * 0.3, duration: 0.3, ease: 'power2.out' });
  });
  btn.addEventListener('mouseleave', () => {
    gsap.to(btn, { x: 0, y: 0, duration: 0.5, ease: 'elastic.out(1, 0.3)' });
  });
});
```

## 4. Parallax layers (simple DOM-based, no 3D needed)

```js
gsap.utils.toArray('[data-parallax-speed]').forEach((el) => {
  const speed = parseFloat(el.dataset.parallaxSpeed); // e.g. 0.3 = slower than scroll, 1.5 = faster
  gsap.to(el, {
    y: () => (1 - speed) * ScrollTrigger.maxScroll(window) * -0.1,
    ease: 'none',
    scrollTrigger: { trigger: el, start: 'top bottom', end: 'bottom top', scrub: true },
  });
});
```

## 5. Text/number counter reveal (used for stats, chapter numbers like the reference site's "01 / 11")

```js
function animateChapterCounter(el, targetChapter, totalChapters) {
  gsap.to(el, {
    innerText: targetChapter,
    duration: 0.4,
    snap: { innerText: 1 },
    onUpdate: function () {
      el.innerText = String(Math.round(this.targets()[0].innerText)).padStart(2, '0');
    },
  });
}
```

## 6. Section nav that highlights current chapter based on scroll position

```js
gsap.utils.toArray('.chapter').forEach((chapter, i) => {
  ScrollTrigger.create({
    trigger: chapter,
    start: 'top center',
    end: 'bottom center',
    onEnter: () => setActiveNavItem(i),
    onEnterBack: () => setActiveNavItem(i),
  });
});
```

## General taste notes

- Stagger and easing choices carry as much of the "premium" feel as the visuals do — favor
  `power3`/`power4` out-easing for reveals (fast start, gentle settle) over linear or basic
  ease-in-out, and use `elastic`/`back` easing sparingly, only for small playful UI elements
  (buttons, cursor), not large layout transitions where it reads as bouncy/cheap.
- Respect `prefers-reduced-motion` (see `performance-and-fallbacks.md`) — disable parallax,
  cursor-follow, and large scroll-driven camera moves for users who've requested reduced motion,
  while still delivering the content itself without motion gating access to it.
