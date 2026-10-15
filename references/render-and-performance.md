# Rendering, performance and delivery

## Render settings (YouTube)

`renderMedia({codec:'h264', pixelFormat:'yuv420p', colorSpace:'bt709', crf:16, x264Preset:'medium', audioCodec:'aac', audioBitrate:'192k', imageFormat:'jpeg', jpegQuality:95, chromiumOptions:{gl:'angle'}, concurrency})`, then `ffmpeg -c:v copy -c:a aac -ar 48000 -movflags +faststart`. Without `colorSpace:'bt709'` JPEG frames produce `yuvj420p` (wrong). Draft: `scale:0.5`, crf 26, veryfast.

Measured on M3/8 GB: default GL 231 ms/frame → ANGLE 156 ms; VideoToolbox no faster and 3.6× bigger files. Heavy scenes (hundreds of textured objects, many filters) can reach 250–340 ms/frame — plan ≈ 7–25 min per 2.5 min video.

## Disk

Each Remotion `bundle()` copies `public/` into the OS temp dir (~600 MB here, voices included) and is NOT cleaned up automatically. Delete the bundle folder in a `finally` after rendering or taking stills. Check free space (`df -h`) before long batches, and check that no `remotion-*` folders pile up in the temp dir.

## Memory

~2.6 GB per Chromium tab. `concurrency = min(cpus-2, floor(totalmem / 2.6 GB))`. Don't run stills/previews in parallel with a heavy render on small machines. Watch `sysctl vm.swapusage`; if swap explodes, kill (`pkill -f render`, `pkill -9 -f headless`) and relaunch lower.

## Orchestration

- `produce.sh <saga> N`: full pipeline (which includes the mandatory thumbnail) → `output/<saga>/<id>/render.log` with `exit=` marker.
- Queue sequentially: `until grep -q "exit=" output/<saga>/ep-PREV/render.log; do sleep 5; done; bash scripts/produce.sh <saga> N` in background; the next job bundles the latest code.
- Monitor progress by grepping `%` lines; notify the user per finished video (desktop notification if they asked).
- If a queued job fails validation, fix, re-validate everything, and re-queue at the end.

## Deliverables per video

`<id>.mp4`, `<id>.<lang>_<REGION>.srt`, `<id>-miniatura.png` — **mandatory** (1280×720 big title + mascot + key visual; shorts: 1080×1920 cover, essentials in the central area). Validation fails if the video has no registered thumbnail composition; the episode pipeline renders it and fails if it's missing, `<id>-audio-maestro.wav`, `<id>-voz.wav`, `<id>-palabras.json` (global word timings), `<id>-timeline-eventos.json` (frames of every event), `qa-automatico.json`, `render.log`, stills.

## Upload metadata

Generate titles (keywords first, ≤ 70 visible chars, emoji ok), descriptions (hook, what you'll learn ✅, how to use, **chapters from real timestamps** — first `0:00`, each ≥ 10 s, next episode, playlist, hashtags), 3–5 hashtags, 15 tags ≤ 500 chars; CSV/JSON for bulk. Kids content: "made for kids" (COPPA) — mention its monetization effects. Never promise "100% SEO".
