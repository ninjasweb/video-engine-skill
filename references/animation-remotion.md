# Animation architecture (Remotion + SVG)

## Layers

```
src/engine/anim.ts      easing (inOut, out, back, spring), progress(), pulse(), seeded rng, samplePath(moves, t)
src/engine/scene.ts     SceneData types, makeCtx(timeline) → at()/sequence()/start()/end()/wait(), inspect(scene, t)
src/engine/subtitles.ts cues, keyword kinds, SRT
src/engine/saga.ts                  SagaModule contract (renderer + scenes + mandatory thumbnails); engine never imports sagas/styles
src/styles/<style>/Crayon.tsx       style primitives (Shape, Stroke, Text, filters); Props.tsx: props library (objects/decor)
src/styles/<style>/character/*      rig + poses + blink (engine stores poses as generic {t, pose: string})
src/styles/<style>/kit.ts           authoring helpers
src/styles/<style>/Video.tsx        makeVideo(scenes, {keywords}) compositor: background → decor → objects → links/rings/labels/numerals → character → overlays → subtitles; audio tracks
sagas/<saga>/scenes/<id>.ts         one declarative builder per video; scenes/index.ts registers scenes + thumbnails
src/sagas.ts · src/Root.tsx         registry of sagas; Remotion root generated from it (<saga>-<id>, thumb-<saga>-<id>)
```

## SceneData (all times in seconds, derived from word anchors)

- `objects[]`: `{id, kind, group, countable, scale, appear, appearDur, remove, moves[]}` — `moves[0]` is the initial position; later moves `{t, x, y, dur, arc}`.
- `labels[]` (text attached to an object), `rings[]` (highlight around object or x/y), `numerals[]` (free text/numbers), `links[]` (pair loops).
- `poses[]` `{t, pose}`; `sfx[]` `{t, name, gain}`; `decor[]` `{kind, appear, remove, props}` (title, mats, containers, number line, arrows, cards, markers, wait timer, end card).
- Optional: `characterMoves` (character walks/jumps), `numberLine`, `answerGroup` (answer is a group count, `'!group'` = anticipated/not yet visible).

## Authoring kit (pattern)

```ts
const k = kit(ctx);
const ids = k.many('apple', 'basket1', at('01','four apples',{word:'four'}) - 0.2, row(4, 1000, 740, 140));
k.takeOut(ids[3], at('01','we eat one',{word:'eat'}), {x: 2050, y: 600});   // leaves the counted area
k.transfer(id, t, dest, 'plate2');                                          // regroup (share out)
k.count(ids, ctx.sequence('03', ['one','two','three']), clearTime);        // label + pop per spoken number
k.eq('03', 'four minus one is three', ['4','−','1','=','3'], end);          // symbol per spoken word
k.check(t); k.pose(t, 'joy'); k.tag('10', x, y, start, end);
return k.finish({title, subtitle, next, nextSub, titleUntil, extra});
```

Rules: anchor to words; clear transient text before question waits; give every created object a `remove` at scene changes; keep objects still during waits; place things inside the layout grid (content zone above the subtitle band).

## Character rig

Parts: body (shape with texture + darker side for volume), face (eyes with blink/pupils/lids, brows, cheeks, mouth), two-segment arms (shoulder angle + elbow bend, mitten hands), legs + shoes. Pose = numeric params (lean, bob, squash, arm angles, look direction, eye openness, happy eyes, brow, smile, mouth open/round, wave, hop, sway, extras like "?" / sparks / shock rays). Blend poses with eased interpolation over ~0.35 s; overlay procedural cycles (breathing, blink schedule from seeded rng, wave oscillation, hop). Mouth opens with the voice envelope (no lip-sync needed). Library: idle, warm, wave, point, curious, think, wait, surprise, joy.

For other styles, keep the rig API and swap the drawing (a robot, an animal, a blob, a presenter card).

## Motion language

- Appear: draw-on (outline → fill scribble → details) + back-ease pop; sound: crayon/pencil + pop.
- Count: bounce + label pop above the object at the spoken word (+ optional ring).
- Move/regroup: ease-in-out with a small arc; whoosh.
- Reveal/confirm: big number or equation symbol-by-symbol; sparkle; confirmation = check chime + joy pose.
- Waits: freeze objects, character "wait" pose (gentle sway, blinking), a non-countable progress indicator (hourglass + filling arc). No hints, no numbers, no SFX.

## Preview

Remotion Studio (props editor) + a small `@remotion/player` page with block/wait navigation buttons, time/frame readout and per-track volume sliders; same component as the render.
