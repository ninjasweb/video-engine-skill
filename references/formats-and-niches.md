# Formats and niches

## Formats

| Format | Size | Notes |
|---|---|---|
| YouTube long | 1920×1080, 30 fps | intro ≤ 3 s, end screen 5–20 s, chapters, burned captions optional + SRT |
| Shorts / Reels / TikTok | 1080×1920, 30 fps | ≤ 60–90 s; hook in 1–2 s; captions centered upper-middle; keep UI safe areas (top ~12%, bottom ~20%, right ~15%); loopable ending |
| Square feed | 1080×1080 | captions bottom third; large type |
| Course/lesson | 1920×1080 | slower pacing, pauses, recap slides, downloadable SRT |
| Ad / product | 16:9 + 9:16 + 1:1 variants | brand kit, CTA end card, 6/15/30 s cuts |

Make width/height/fps come from the script (`video` field) and compute layout zones from them; never hardcode 1920×1080 in scene files when multiple formats are expected (use relative helpers).

Proven pattern: one `layoutFor(w, h)` module returns subtitle band (bottom offset, font size, chars/line), `sceneBottom`, `safeTop`, `safeRight`; the composition takes width/height from the timeline via `calculateMetadata`; the SVG `viewBox` uses them; backdrop decoration has a per-orientation table; the validator enforces the zones (vertical: objects y ∈ [230, 1340], x ≤ 900; subtitles 66 px, 22 chars/line, bottom offset 400 px). Shorts use their own script folder + schema, short intro (≈0.35 s — first word immediately), gaps ≈0.35 s and a 3 s outro/CTA. Vertical scenes are authored per format (not auto-reflowed); raised arms, pose marks ("?") and pointing hands take extra room — check them on the contact sheet (5 columns of 360×640).

## Niche playbooks (starting points, adapt to references)

- **Kids education**: mascot, question → wait → check loop, numbers revealed only when spoken, pauses 5–10 s, burned captions with keyword highlight, "made for kids".
- **Explainer / edu for adults**: kinetic typography + diagrams, problem → insight → example → recap, B-roll-like illustrated scenes, chapters.
- **Storytelling / history / true stories**: narrator voice with emotional tags, parallax paper layers, map/route animations, slower music, captions for sound-off viewing.
- **Finance / data**: animated charts (draw-on lines/bars), numbers counting up synced to words, disclaimers.
- **Product / SaaS**: UI mockups sliding in, cursor animations, feature callouts, CTA.
- **Language learning**: word cards appearing on pronunciation, bilingual captions, repetition pauses.
- **News / facts shorts**: bold headline cards, fast cuts, source lower-thirds, big captions.
- **Meditation / wellness**: slow gradients, breathing circles synced to voice, ambient pads, minimal text.

For each: define the block roles (hook, teach, example, question/wait, check, recap, CTA), the visual grammar per role, and the validator rules that matter (e.g. finance: numbers shown only after spoken; kids: waits frozen; shorts: first word ≤ 0.5 s, caption always visible).
