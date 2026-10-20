---
name: motion-composition
description: Conventions for this Remotion project (motion graphics in React) — adding/editing compositions in src/compositions, wiring them into src/Root.tsx, rendering (opaque/transparent/PNG-sequence, portrait variants), fonts, sound effects, and the derived-asset pipeline in tools/. Use whenever the user asks to add a graphic/intro/stinger/title, change an existing composition, render output, add SFX, or regenerate art from source assets in this project.
---

# Motion (Remotion) project conventions

A Remotion project: motion graphics defined as React components, previewed
in Remotion Studio, rendered to video. Every composition lives in
`src/compositions/`, registered in `src/Root.tsx`.

## Global config

Everything reads from `src/config.ts` — never hardcode width/height/fps/durations in a composition.

- `VIDEO_WIDTH` / `VIDEO_HEIGHT` / `VIDEO_FPS` — main 16:9 canvas.
- `VIDEO_WIDTH_PORTRAIT` / `VIDEO_HEIGHT_PORTRAIT` — 9:16 canvas for Reels/TikTok/Shorts.
- One `DURATION_<NAME>` constant per composition, expressed as `N * VIDEO_FPS` (seconds, not raw frames) so it stays correct if fps changes. Add a comment above each new one explaining *what* the duration covers and, if the composition has internal beat timings (e.g. "shield slams in at frame 62"), which frame those beats land on and what changing the duration does/doesn't affect (usually: stretches the hold, not the build).
- `TITLE_SAFE_MARGIN` — keep on-screen text inside this margin.

## Adding a new composition

1. Create `src/compositions/MyGraphic.tsx` (or a subfolder like `crest/`, `smp4/` if it's a multi-file family — see below). Shape it like an existing file:
   - a `zod` schema describing every prop that should be editable in Studio
   - a `defaultProps` object matching that schema
   - the component itself, driven by `useCurrentFrame()` / `useVideoConfig()` — never CSS animations or timers
2. Add a `DURATION_MY_GRAPHIC` constant to `src/config.ts`.
3. Register it in `src/Root.tsx`:
   ```tsx
   import { MyGraphic, myGraphicSchema, myGraphicDefaultProps } from "./compositions/MyGraphic";

   <Composition
     id="MyGraphic"
     component={MyGraphic}
     durationInFrames={DURATION_MY_GRAPHIC}
     fps={VIDEO_FPS}
     width={VIDEO_WIDTH}
     height={VIDEO_HEIGHT}
     schema={myGraphicSchema}
     defaultProps={myGraphicDefaultProps}
   />
   ```
   The `id` is what `remotion render` targets. Add a short JSX comment above the `<Composition>` explaining anything non-obvious: whether it's a transparent overlay, a seamless loop, a stinger meant to cut on a specific frame, etc. — match the style already in `Root.tsx`.
4. If the graphic makes sense for social (Reels/TikTok/Shorts), add a second `<Composition id="MyGraphicPortrait">` reusing the same `component`/`schema` with `VIDEO_WIDTH_PORTRAIT`/`VIDEO_HEIGHT_PORTRAIT`. Prefer making the component itself aspect-ratio-aware (measurements as fractions/multiples of canvas size) over writing a separate portrait layout — that's the pattern every existing portrait variant follows.
5. For a family of related compositions (a reveal + a looping parallax + a short stinger, e.g. `crest/`, `smp4/`, `raft/`), put shared geometry/asset-path/animation-helper code in that folder's own `shared.tsx` and import it from each variant, rather than duplicating.

## Transparency rule

A composition renders with a real alpha channel **only if nothing in its
component tree ever paints an opaque background** — no opaque
`backgroundColor` anywhere, including on a parent wrapper. If a composition
composites opaque art (photos, `mix-blend-mode`, a painted backdrop), don't
offer it transparent — say so in a comment, the way `DialogueScene` and
`RaftProject` do.

## Rendering

Target a composition `id` from `Root.tsx`.

```bash
# Standard opaque MP4
npx remotion render MyGraphic out/MyGraphic.mp4 --codec=h264 --crf=18

# Transparent overlay (real alpha channel, for dropping over footage)
npx remotion render MyGraphic out/MyGraphic.mov --codec=prores --prores-profile=4444 --pixel-format=yuva444p10le --image-format=png

# Transparent PNG sequence (for tools that don't read ProRes)
npx remotion render MyGraphic out/MyGraphic/frame-%04d.png --image-format=png
```

`out/` is git-ignored — nothing rendered belongs in version control.

## Fonts

Load Google Fonts on demand rather than relying on system fonts (a render
server/CI machine won't have them installed):

```tsx
import { loadFont } from "@remotion/google-fonts/Inter";
const { fontFamily } = loadFont();
```

See remotion.dev/fonts for available families/weights. `KineticText.tsx` is the reference example.

## Sound effects

Play audio via `<Audio>` inside a `<Sequence>` so playback start is frame-accurate:

```tsx
import { Audio, Sequence, staticFile } from "remotion";

<Sequence from={10}>
  <Audio src={staticFile("whoosh.mp3")} />
</Sequence>
```

`from={N}` trims leading silence without re-exporting the file.

To generate a new cue set instead of sourcing audio: write a
`tools/make-<name>-sfx.js` script using the synthesis helpers in
`tools/audio.js` (oscillators, filters, reverb, WAV writer) — follow the
shape of an existing one (e.g. `tools/make-toon-sfx.js`,
`tools/make-raft-sfx.js`). Output goes to `public/<name>/sfx/`. Check a cue
set without listening to every file: `node tools/inspect-sfx.js public/<name>/sfx` prints a loudness/brightness sketch per file.

## Derived-asset pipeline

**Nothing generated is committed** — every file under `public/*/layers/`,
`public/*/sfx/`, or similar must be reproducible from the delivered source
art (`public/*/source/...`) by re-running the relevant `tools/*.js` script.
When adding a new logo/badge that needs the same treatment:

1. Add a `tools/<name>-alpha.js` (knock the paper/background out of the delivered render) if the source has a flat background to remove.
2. Add a `tools/segment-<name>.js` (cut the art into the layers a composition will animate independently) if the piece has multiple moving parts. Follow the region-labeling approach in `tools/segment-logo.js`/`tools/segment-smp4.js` — label enclosed regions by connected components rather than flood-filling from the border, since pockets fully enclosed by artwork can't be reached that way.
3. Write each layer's content box / anchor points to a `layers.json` next to the layers, and have the composition read its `ORIGIN` constants from that file rather than hardcoding pixel offsets.
4. Document the regenerate commands in the README's fenced block (the "Nothing generated is committed" section), same as the existing scripts.

## Before finishing

Run `npm run lint` (`eslint src && tsc`) after touching anything in `src/`.
