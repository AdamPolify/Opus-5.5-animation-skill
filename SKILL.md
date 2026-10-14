---
name: ticket-style-motion
description: Build polished animated brand pieces (3D ticket launches, animated profiles/banners, animated Figma frames) as HTML and export them to smooth MP4.
---

# Ticket-style motion: animated brand pieces → HTML + MP4

Use this when the user wants a design (Figma frame, ticket, logo, profile/banner, bento of cards) brought to life. Typical asks: "animate this", "make it like the ticket one", "each section should be animated", "make it into an mp4".

Deliver two things unless told otherwise:
1. A self-contained animated HTML page, published as an Artifact (images go in `files`).
2. An MP4 rendered frame-by-frame from that page (never screen-recorded), sent with SendUserFile.

---

## 1. Workflow

1. **Get the real assets first.** Never redraw something that exists as an image.
2. **Lay out the stage at the design's native size.** Use the exact px from the design, and scale the whole stage to fit the window.
3. **Plan motion per section from what's actually in the design.** Each section gets three things:
   - an *entrance* (once);
   - one *move* that suits that object (a pack floats, a box rattles, a chart draws);
   - a quiet *ambient loop*.
4. **Build with compositor-only animation** (see Performance).
5. **Add a deterministic `__frame(t)` hook** so the same page can be scrubbed for video.
6. **Verify with contact sheets.** Render several timestamps, crop the interesting parts, and look at them before publishing.
7. **Publish the artifact, render the MP4, send it.** Explain the result in plain words; mention the caveats that matter.

First-reply rule: say what you're about to do, then start. Ask only when an instruction is genuinely ambiguous *and* expensive to redo. Example: "use the logo background" could mean three different assets, so ask with options.

Scope rule: change only the project the user is talking about. If they say "not for X", fully revert X (page and video) before doing the real request.

---

## 2. Getting assets out of Figma

- **Layout and values.** Use `use_figma` to read node trees, boxes, fills, strokes, effects (inner shadows, blur radii), fonts and letter-spacing.
  - Map Figma INNER_SHADOW to CSS `inset 0 <y>px <radius>px rgba(...)`. The Figma radius is the CSS blur.
- **Exact placement.** Compute each image's `absoluteRenderBounds` minus the frame's `absoluteBoundingBox`.
  - That gives the frame-relative x/y/w/h; place assets with those numbers.
  - Note when the bbox ≠ render bounds (rotated or masked images). Position by the *bbox* if the screenshot is the full unclipped image.
- **Raster images.** `get_screenshot` (with `contentsOnly:true`, `enableBase64Response:true`) saves a **transparent RGBA PNG** to the session's `tool-results/` folder. Copy it out immediately.
  - It's 1x max: `maxDimension` won't upscale.
  - Masked or clipped images come back pre-clipped. Check the alpha bbox with PIL to see which edges are cut.
- **Vectors.** `exportAsync({format:'SVG_STRING'})` returns SVG strings for icons, wordmarks and charts.
  - Parse them out of the transcript `.jsonl` with a script and write them to files. Don't retype path data.
  - Inline them via a template plus a build step (`{{name}}` placeholders).
- **Fallback for big binaries.** Use `use_figma` export → `figma.base64Encode` → 19 000-char chunks per call (the return cap is about 20 KB).
  - Extract the chunks from the transcript with a regex tolerant of escaped quotes.
  - **Save to disk right after fetching.** The transcript gets compacted or rotated and the chunks vanish.
- **Figma asset URLs** (`figma.com/api/mcp/asset/...`) are usually blocked from the shell. Don't rely on curl.
- **Fonts.** Load the design fonts from Google Fonts in the page.
  - For local verification renders, install the font files (e.g. `npm i geist`, copy the TTFs to `~/.fonts`, run `fc-cache`). Otherwise screenshots use fallback fonts.

---

## 3. Page skeleton

```html
<div class="stage" id="stage">…sections…</div>
<script>
  const stage = document.getElementById('stage');
  const fit = () => {
    const s = Math.min(innerWidth / W, innerHeight / H);
    stage.style.transform = `translate(-50%,-50%) scale(${s})`;
  };
  addEventListener('resize', fit);
  fit();
</script>
```

- `.stage{position:absolute;left:50%;top:50%;width:Wpx;height:Hpx;transform-origin:50% 50%;overflow:hidden}`.
- Keep a **template.html + build script** that inlines the SVGs and writes `index.html`. Re-run the build after every template edit.
- Add a small **↻ Replay** button. It cancels and replays all animations (`document.getAnimations()`) and restarts JS timers. Hide it in capture mode.
- Respect `prefers-reduced-motion`.
- Use a seeded RNG (`seed = seed*16807 % 2147483647`) for anything random, so every loop and every export is identical.

---

## 4. Techniques

**Rule:** pick motion from what's actually in the design and what the user asks for. These are proven ways to do specific things when they fit. They are not a checklist to add to every piece; repeating the same moves everywhere makes every project look the same.

### Basics (most pieces)

- **Staggered entrances,** 0.12–0.15 s apart.
  - Cards: `opacity 0 → 1` + `translateY(46px) scale(.955) → none`, cubic-bezier(.2,.75,.2,1), ~0.95 s.
  - Text: rises 12 px and fades.
  - Small items: pop in with overshoot (`cubic-bezier(.3,1.6,.5,1)` from `scale(.4)`).
- **Wordmark reveal:** clip-path wipe `inset(0 100% 0 0) → inset(0)`.
- **SVG lines and charts:** `pathLength="1"` + `stroke-dasharray:1; stroke-dashoffset:1 → 0`, then fade the area fill in after.
- **Ambient loops,** subtle and slow:
  - float (translateY 5–10 px + ±1° rotation, 5–6 s);
  - breathing glow (opacity);
  - occasional light sweep across a card;
  - a dot travelling a path (`offset-path` + `offset-distance`).
- **Clipped images that float:** only move them *into* the clip, or scale them slightly, so cut edges never show.

### Bending paper: the 3D ticket launch

- **Scene.** A `perspective` scene with a `preserve-3d` world. A slot in a surface, with a dark "inside" plane at negative Z seen through the hole.
- **Strips.** Build the ticket from **thin horizontal rows** (~10 px), each with the same background image offset by its row y.
  - `bend(thetaAt(s))` walks down the paper, accumulating `y += RS*cos θ, z += RS*sin θ`.
  - Each row gets `translate3d(0,y,z) rotateX(θ)`, so the paper curls without stretching. Shade rows by angle with a small `brightness()`.
- **Sequence:**
  1. **Print:** a receipt-printer feed in ~6 quick jerks (step-eased). Rows still inside the slot are hidden or clip-pathed.
  2. **Gust:** curvature `amt·(s/880)^1.35`; the top stays at the slot while the bottom lifts toward the viewer.
  3. **Fly at camera:** easeIn on z toward `P*(1-1/scale)`, with a sine wobble on rx/rz and the bend relaxing to flat just before impact.
  4. **Impact:** a radial white flash, an expanding ring, a ~320 ms screen shake and a glare sheen masked to the ticket image. Then a damped settle: `exp(-6k)·sin` on rotation/scale.
  5. **Exit:** peel. The bottom curls away first, then it drops out of frame. Loop.

### Rolling numbers (odometer)

- Each digit is a column of 0–9 repeated 4× inside a `1.15em` overflow-hidden box. Animate `translateY(-idx·1.15em)` with the Web Animations API.
- Always roll **up** (wrap by adding 10). Intro: extra spins for the rightmost digits, staggered.
- Optional live ticks afterwards. Use `tabular-nums`.

### Light travelling along stripes baked into a flat image

Use when the glow or stripes are part of the image itself. Don't add new fake rings.

1. In Python, mask the stripe pixels by colour and brightness, and exclude logos with a soft ellipse.
2. Write `*_base.png` (stripes dimmed toward their tint) and `*_light.png` (stripes only, plus a tinted bloom).
3. Put the light layer in a square `.lit` that rotates with a **conic-gradient mask** (two comet arcs with tails).
4. Its inner image counter-rotates around the same centre, so only the mask spins and the existing stripes light up in sequence.

### Flicker or glow without lag

- Use a pre-blurred static copy (`filter` set once) whose **opacity** flickers, plus an optional brightened copy for flashes.
- Never animate `filter`.

### Part of a single image moving (e.g. a lid lifting)

- Duplicate the image and clip the moving part with a `clip-path` polygon. Animate it with a hinged `transform-origin`, synced to the body's rattle.
- Cover the seam with a glowing polyline and light beams whose opacity peaks with each movement.
- Scale the masked image ~1.09 so movement never reveals clipped edges.

### Surface shine

- A container masked by the image (`mask: url(img.png)`), with a child gradient bar moved by `translateX`.

---

## 5. Performance ("it's laggy" fixes)

Animate **only `transform` and `opacity`**, and add `will-change` on those layers. Known lag sources and their replacements:

| Laggy | Replace with |
|---|---|
| Animated `filter` (drop-shadow, blur, brightness) | Static pre-filtered copy + opacity |
| `background-position` sweeps | Child element moved with `translateX` |
| `mix-blend-mode` on full-screen or animated layers | Normal blending, lower alpha |
| Blur in card entrances | Opacity + transform only |
| Several huge drifting gradient blobs | One static background layer |
| Full-screen animated grain | Static noise baked into the stage background |
| CSS masks on many spinning elements | One static SVG per ring, rotated |
| `requestAnimationFrame` loop writing text forever | Event-driven updates (WAAPI / timers) |

Measure with a Playwright fps probe. Toggle candidate layers off with an injected `<style>` and compare rAF counts over 2 s. Headless numbers are pessimistic, but the *relative* gains are real.

---

## 6. Deterministic capture → MP4

```js
window.__frame = (t) => {
  document.getAnimations().forEach((a) => { a.pause(); a.currentTime = t * 1000; });
  // JS-driven bits as pure functions of t (e.g. odometer positions from a fixed event schedule)
};
if (location.hash.includes('capture')) document.documentElement.classList.add('capture');
```

- In capture mode, don't start live timers. Express timer-driven effects (number ticks, tweens) as `f(t)` with a fixed schedule.
- Playwright: set the viewport to the stage size (and `device_scale_factor` for a hi-dpi card). For each frame `i`, call `__frame(i/30)`, then save a JPEG screenshot (quality 95). Then run:

```bash
ffmpeg -framerate 30 -i f%04d.jpg -c:v libx264 -pix_fmt yuv420p -crf 17 -preset slow -movflags +faststart out.mp4
```
- **Videos inside the design** (e.g. a 3D logo as an avatar). Overlay a `<video autoplay muted loop playsinline>` in the slot on the page. Hide it in capture mode, then composite the original source in ffmpeg with a rounded-rect alpha mask:

```bash
-filter_complex "[1:v]scale=S:S,format=rgba[v];[2:v]format=gray[m];[v][m]alphamerge[a];[0:v][a]overlay=X:Y"
```
  - Use `-stream_loop` for repeats; make the length a multiple of the source loop.
  - If the rest of the card is static, use `-loop 1 -i card.png` instead of rendering frames.
- Serve the folder with `setsid nohup python3 -m http.server PORT &`. Servers die between turns, so check with curl and restart.
- 1080p × 13 s ≈ 1.5 min of rendering.

---

## 7. Image surgery for backgrounds and banners

- **Removing an object from a video frame.**
  - A temporal median fails when the camera moves, and plain `cv2.inpaint` smears.
  - What works: a `cv2.seamlessClone` of the *same frame shifted horizontally* (`np.roll`) into a dilated polygon mask over the object. Try a few shifts and pick the cleanest.
- **HUD text and frame lines.** Same clone trick, or crop inside them.
- **Out-of-focus backdrop.** Large Gaussian blur, a dim (~0.68×) and a baked vignette, all in one JPG. No overlay layers.
- **Sharp banner from a square frame.** Crop the largest 3:1 strip, Lanczos-upscale and add a light unsharp mask. Tell the user it's upscaled.

---

## 8. Verification checklist

- [ ] Contact sheet at entrance (≈0.7 s, 1.4 s), mid-intro (≈2.5 s) and settled state (≈7 s), compared to the Figma screenshot side by side.
- [ ] Crops of each section's move at its peak.
- [ ] No clipped edges revealed by floats; nothing spilling outside its banner or card (clip with `clipPath` if needed).
- [ ] Video: ffprobe the duration and size; a few frames extracted and viewed.
- [ ] Keep backups of replaced assets (`raw/…_v1.jpg`) so you can revert.

## 9. Reporting back

- Name what changed per section in plain words (motion, not CSS).
- State real limits honestly. For example: "the stripes are baked into a flat image, so the light travels along them rather than the stripes themselves rotating"; or "the lid is cut by hand".
- Offer one concrete next step (longer cut, vertical/square export, seamless loop).
