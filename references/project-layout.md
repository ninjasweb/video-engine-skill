# Project layout: many videos, one engine

Every project is organized from the first video as if it will host many. Clients keep adding videos, often
in different looks, and a flat single-video repo has to be refactored later.

Two proven variants:

- **Series** (episodes sharing a character, voice and format). Use `sagas/<saga>/{saga.json, episodes/, scenes/, schemas/}`.
  Documented in a `SAGAS.md`.
- **Independent videos** (one-off explainers, each with its own topic, supplied voice, assets and maybe its own look).
  Use `videos/<slug>/`, documented in a `VIDEOS.md`.

## Independent videos layout

```
videos/_template/            skeleton for `npm run video -- new` (slugs starting with "_" are ignored)
videos/<slug>/               everything specific to one video
  video.json                 identity + pipeline settings (title, locale, style, audio, formats, fps/lead/tail,
                             must_hear terms, spoken_numbers digit→words map for the validator, default still instants)
  guion.md|.txt, <voice>.mp3 sources the client drops in (the folder owns them)
  script.json                derived blocks (md scripts: only the narration sections are spoken; visual notes kept per section)
  data/words.raw.json        alignment
  generated/timeline.json    imported statically by index.ts (placeholder with no blocks until the timeline step)
  scene.ts                   visual script anchored to spoken words
  Thumbnail.tsx, youtube.ts  mandatory thumbnail; upload copy (chapters come from the scene)
  index.ts                   the VideoModule: {config, timeline, build, Video, Thumbnail}
  style/                     per-video element kinds / props
  assets/                    per-video media, synced to public/videos/<slug>/assets/
  references/                research, sources, fidelity notes
src/engine/                  style-agnostic core + the contract (video.ts: VideoModule, buildScene, naming)
src/styles/<style>/          reusable look: theme, renderers, makeVideo(), kit (scene helpers), STYLE.md
src/videos.ts, src/Root.tsx  registry → compositions <slug>-<format>, thumb-<slug>-<format>
public/styles/<style>/       style sounds/music; public/fonts shared; public/videos/<slug>/ DERIVED
output/<slug>/               deliverables named <slug>-16x9.mp4, <slug>.<locale>.srt, <slug>-miniatura-16x9.png…
```

Every script takes `--video <slug>` (or the `VIDEO` env var). It may be omitted only while one video exists, and an
unknown or missing slug fails with the list of videos.

## Three levels of reuse for the look

1. **Reuse a style:** `"style": "<style>"` and `makeVideo()`.
2. **Extend it per video:** new element kinds in `videos/<slug>/style/`, which are pure `(e, t)` renderers following the
   style's element contract. Pass them as `makeVideo({kinds})` and draw them in the thumbnail with
   `<ElementView kinds>`. They get validated like built-in kinds. Promote a kind to the style once a second video needs it.
3. **New style:** `new --style <name>` copies the base style (code + public assets) as a starting point to restyle.

The rule is to never change a shared style for one video in a way that changes the others. If you must change it,
re-run the other videos' stills.

## Remotion specifics that shape the layout

- `staticFile()` only reaches `public/`. Keep per-video sources in the video folder and copy them (new/changed files only)
  into `public/videos/<slug>/` before every bundle and before the Studio. The timeline step writes the normalized master
  voice there too. Don't symlink: `bundle()` copies `public/` and symlinks are fragile.
- Timelines are imported statically (Node validator + bundle share them). A new video therefore needs a placeholder
  timeline with `blocks: []`, and `Root` / `buildScene` skip it or fail with "run the timeline first".
- Keep one registry file (`src/videos.ts`) of static imports. Node scripts can't use webpack's `require.context`, so the
  `new` command edits the registry.
- Naming (composition ids, output names, thumbnail sizes) lives in one module that `Root` and every script import.
  Don't hardcode ids in scripts.
- A bundle includes every video's public files. That's fine for a handful of videos; for dozens, bundle per video or keep
  voices compressed.

## Migrating a flat repo (one video) to this layout

1. **Baseline first.** Capture full-resolution stills at about 8 instants per format, the thumbnails, every JSON
   deliverable (timeline, events, words, QA, metadata), the SRT and the master voice md5.
2. Move files with `mv` (never delete; park leftovers in `_legacy/` for the user to review). Extract the scene helpers
   into the style's `kit.ts`, the renderer into `makeVideo()`, and the per-video constants (key terms, number words,
   still instants, copy) into `video.json` / `youtube.ts`.
3. Re-run the pipeline. Everything must be **byte-identical**: stills md5, thumbnails, JSON and SRT. The only expected
   difference is the voice path in `timeline.json`. Then render a short draft sample to prove the audio paths resolve,
   and compare its loudness with the same window of the old render.
4. Scaffold the next video with the new command on the folder the client prepared, and typecheck.
