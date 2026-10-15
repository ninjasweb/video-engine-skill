# Textures and looks with SVG filters (seeded, stable)

All filters use `feTurbulence` with a **fixed seed** and are applied on groups whose transform is the object's own — the noise lives in object space, so it moves with the object and never flickers. Measure: `ffmpeg … tblend=all_mode=difference,signalstats` over a static crop → YAVG ≈ 0.

## Crayon / wax (worked example)

- Fill: clip to shape → light wash + 3 zigzag hatch passes (main angle, +68°, −28°; widths ~1.25×/0.7×/0.55× spacing) + a thick inner stroke of the darker color (accumulated wax at edges).
- Filter `wax`: turbulence(0.035, 2 oct) → displacement 6 px (rough edges) → high-freq turbulence alpha thresholded (`values="0 0 0 0 0 … 0 0 0 -4.6 3.25"`) as a mask → paper gaps.
- Outline: stroke with `line` filter (displacement 3.5, finer grain).
- Draw-on: `pathLength=1`, `strokeDasharray="p 2"` for outline and each hatch pass in sequence.

## Other looks (same primitives, different parameters)

| Look | Fill | Edge | Filter |
|---|---|---|---|
| Clean flat vector | solid | crisp stroke or none | none |
| Pencil sketch | sparse hatching, low opacity, graphite gray | double wobbly strokes | small displacement, fine grain |
| Marker | solid with streak passes, 90% opacity | slightly bleeding | tiny blur + displacement |
| Watercolor | radial gradient + blotch noise mask | darker pooled edge (inner stroke, blur) | turbulence(0.01) displacement + feGaussianBlur 1–2 |
| Paper cut-out | solid | none | drop shadow (offset, blur) + paper grain overlay |
| Chalkboard | dry-brush hatch, white/pastels | rough | strong grain mask, dark green/charcoal bg |
| Halftone / comic | solid + dot pattern `<pattern>` | thick ink outline | none |
| Neon | none | glowing stroke | feGaussianBlur stacked + additive composite |
| Clay/soft 3D | radial gradient highlight + shadow | none | inner shadow via feOffset/feComposite |
| Pixel art | render at low res grid; `shape-rendering: crispEdges` | none | none |

Backgrounds: generate a static PNG (paper grain, gradient, grid, chalkboard) once with numpy/PIL; decorative corner elements drawn once in intro.

## Legibility

- Numbers/text: display font, fill + outline in the background color (paint-order stroke) for contrast; texture only via mild filter.
- Keep text ≥ 50 px at 1080p for kids; subtitle ≥ 52 px bold with 10 px dark outline.
