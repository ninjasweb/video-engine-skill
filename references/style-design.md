# Style design: inferring or proposing a visual identity

The engine is style-agnostic. Before writing any scene, produce a **style bible** (a short doc or `style.json`) and render a 12–15 s sample from a real audio fragment for approval.

## 1. From client references (images, videos, brands, links)

Look at every reference (use the Read tool on images; extract frames from videos with `ffmpeg -ss T -i v.mp4 -frames:v 1 f.png`) and extract:

| Dimension | What to extract | Example (crayon kids) |
|---|---|---|
| Medium / finish | flat vector, crayon, watercolor, paper cut-out, chalkboard, ink, clay/3D-ish, pixel, glass UI, photo collage | wax crayon on ivory paper |
| Palette | 5–8 colors with roles: background, primary, accents, text/ink, success, warning | mint, blue, yellow, orange, red, pink; navy ink |
| Line quality | none / clean / wobbly / sketchy; stroke weight | slightly wobbly, stable, 4–6 px |
| Fill texture | solid, gradient, hatching, grain, halftone, noise | zigzag hatching + wax gaps |
| Background | plain, textured paper, gradient, environment | ivory paper grain + corner scribbles |
| Typography | display + body; rounded/serif/hand; weight | Fredoka (display), Patrick Hand (hand) |
| Character | shape language, proportions, face, limbs, expressions | rounded mint square, big eyes |
| Motion | snappy/bouncy/smooth; overshoot; draw-on; camera | pop with back-ease, draw-on, gentle bob |
| Sound | music genre/tempo, SFX family | soft marimba, crayon scratch, pop, chime |
| Pacing | cuts per minute, pauses, text density | slow, pauses for answers |

Write the bible with **concrete values** (hex codes, font files, px sizes, easing names, BPM, dB levels). Ambiguity here becomes inconsistency across episodes.

## 2. Without references: propose directions

Offer 2–3 contrasting directions matched to niche + audience, each with a one-line pitch, palette, font pair, texture, motion and sound. Heuristics:

- **Kids (3–8)**: warm saturated palette, rounded shapes, hand-drawn texture, big readable numerals, a friendly mascot, slow pace with pauses, soft percussive SFX.
- **Teens/edu explainers**: bold flat colors or notebook/doodle style, fast kinetic typography, meme-aware timing, punchy SFX.
- **Corporate/product**: clean flat vector or glass UI, 1 brand color + neutrals, geometric sans, smooth ease-in-out, subtle whooshes, ambient music.
- **Finance/data**: dark or off-white, high-contrast accent, charts animated with draw-on, mono/sans numerals, restrained sound.
- **Storytelling/history**: paper cut-out or watercolor, parallax layers, serif display, cinematic music, slower camera moves.
- **Science/tech**: blueprint/chalkboard or neon-on-dark, grid backgrounds, diagram draw-on, synth textures.
- **Wellness/lifestyle**: pastel, soft grain, rounded sans, slow floaty motion, ambient pads.
- **Shorts/Reels/TikTok (9:16)**: hook in the first 1–2 s, big centered captions (word-by-word, highlighted), fast cuts every 1–3 s, safe areas for platform UI.

Always ask the user to choose (or confirm the inferred style) before scaling to many videos.

## 3. Turning a style into engine primitives

All looks are built from the same primitives in SVG (see `textures-and-rendering-looks.md`):
- `Shape(d, fill, outline, texture)` — texture = solid | hatching | grain | watercolor bleed | halftone | noise.
- `Stroke(d, width, texture, draw 0..1)` — draw-on via `pathLength=1` + dasharray.
- `Text(text, font, fill, outline, reveal)` — with optional texture filter.
- Character rig — parts + joints + pose params (see `animation-remotion.md`).
- Background — static PNG texture (generated once) + decorative static elements.

Keep **one palette object** and **one filter set** per project so every scene inherits the style.

## 4. Consistency rules

- Same character proportions, colors and outline across all videos; poses change angles, never shapes.
- Same numeral/typography treatment for all on-screen numbers and text.
- A reserved layout grid: character zone, content zone, title/equation zone, subtitle band, platform safe areas.
- A fixed SFX palette mapped to meanings (appear, count, move, reveal, correct) at fixed relative levels.
