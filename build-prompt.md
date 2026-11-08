# Build prompt: browser video clipping editor with one burned-in caption style

You are a coding agent starting cold in a Claude cloud environment. Build the app described here in one pass and ship it as a pull request to `https://github.com/n-eptunium/none-the-weisser.git`. Everything you need is in this document. Decisions below are final unless marked **Assumption**, which you may tune but must report.

## 1. Product

A single-page web app that runs in the user's own browser on their own computer. Flow: open a long local video, mark clips with in/out points, transcribe each clip to captions, correct caption text and timing, export each clip as an MP4 with the captions burned in. Nothing is uploaded anywhere. There is exactly one caption style, defined in section 5. No style picker, no template gallery, no caption-design panel of any kind.

Out of scope. Do not build any of these, even partially: multiple source files, transitions, music or sound effects, B-roll, zoom or punch-in effects, word-by-word highlighting or karaoke colouring, emojis in captions, caption style options or presets, SRT or VTT import, SRT or VTT export, auto-highlight or AI clip selection, colour correction, audio ducking or normalisation, batch export, a server or user accounts, mobile or touch layouts, audio waveform display, filmstrip thumbnails on the timeline, a text-based logo or watermark.

## 2. Architecture decisions

**Stack.** Vite 8, React 19, TypeScript strict, Zustand for state, Vitest for unit tests, Playwright for browser tests. No CSS framework; hand-written CSS with custom properties from section 8. Keep dependencies to: react, react-dom, zustand, mediabunny, @huggingface/transformers, @fontsource/dosis, plus dev tooling.

**Runtime is fully client-side.** No backend, no local helper binary, no ffmpeg.wasm. Media is read, decoded, encoded and muxed with WebCodecs through `mediabunny` (npm, MPL-2.0, use the current 1.65.x line). Reasons: WebCodecs uses the OS hardware or built-in software codecs, so a 1080p clip exports in seconds rather than minutes; mediabunny reads a `File` through `BlobSource`, which slices on demand and never loads the whole file; ffmpeg.wasm would need the whole file in memory and is 10 to 50 times slower.

**Large files.** Playback and scrubbing use a plain `<video>` element pointed at `URL.createObjectURL(file)`. Browsers stream blob URLs from disk with range reads, so a multi-gigabyte file works. Transcription decodes only the audio of the selected clip, never the whole file. Export decodes only the clip's frame range. Exported bytes are held in memory in a `BufferTarget` and saved with an `<a download>` anchor; this bounds a clip to the configurable constant `MAX_CLIP_SECONDS = 180` (**Assumption**: three minutes of 1080p H.264 is roughly 100 to 250 MB, safe in memory on a desktop).

**Supported browsers.** Primary target is Chromium 120 or later on desktop (Chrome, Edge, Brave, Arc). Firefox 130 or later and Safari 26 or later have the required WebCodecs APIs on paper but are not verified by this build; the app shows a plain unsupported notice if `VideoEncoder`, `VideoDecoder`, `AudioDecoder` or `OffscreenCanvas` is missing. The app must be served from `localhost` or HTTPS because WebCodecs is secure-context only; this was confirmed: on a non-secure origin `VideoEncoder` is undefined.

**Codec policy.** Output container is MP4 with `fastStart: 'in-memory'`. Video is H.264 (`avc`) re-encoded through mediabunny with quality `QUALITY_HIGH`, hardware acceleration `no-preference`. Audio is copied from the source without re-encoding whenever mediabunny can copy it (AAC in, AAC out); audio packets are independently decodable so trim precision is one audio frame (about 21 ms at 48 kHz), which is acceptable. If copying is impossible and the browser can encode AAC, re-encode to AAC. Otherwise fall back to Opus in MP4 and show a warning pill on the export (Opus in MP4 plays in Chrome and VLC but not QuickTime). This matters because headless Chromium on Linux has no AAC encoder, while the user's desktop Chrome usually does.

**Preview equals export.** One pure function renders a frame, and both the live preview canvas and the export pipeline call it. See section 6.

**Transcription** runs in a Web Worker with `@huggingface/transformers` 4.x and the model `onnx-community/whisper-base_timestamped`, which is the ONNX export that includes decoder cross-attentions, required for `return_timestamps: 'word'`. See section 7.

**Persistence.** The project (clips, captions, aspect settings) autosaves to IndexedDB keyed by the source file's fingerprint `name + size + lastModified`. Reopening the same file restores the project. There is no project file import or export.

**Undo.** Snapshot undo and redo of the whole project state, 50 steps, Ctrl or Cmd+Z and Shift+Ctrl or Cmd+Z.

## 3. Shipping rules

- Work on a new branch named `feat/clip-editor` off `main`. Commit after each build step in section 10 with a clear message. Push with `git push -u origin feat/clip-editor`.
- Open one pull request against `main`. The description must list what was built, exactly what was verified and how, and a section titled "Not verified in this environment" listing everything you could not test (see section 9). If something does not work, say so in the PR; do not describe it as done.
- Never commit video files, audio files, model weights, `.onnx` files, Playwright traces or screenshots other than the golden images in `tests/golden/`, `node_modules`, or any secrets. Add this `.gitignore` in the first commit:

```
node_modules/
dist/
coverage/
.vite/
test-results/
playwright-report/
tests/fixtures/*.mp4
tests/fixtures/*.webm
tests/fixtures/*.wav
tests/fixtures/*.mp3
tests/e2e/out/
*.mp4
*.mov
*.mkv
*.webm
*.wav
*.mp3
*.onnx
*.bin
*.safetensors
*.gguf
.env
.env.*
*.log
.DS_Store
```

- Include a README with: what the app does, supported browsers, `npm install`, `npm run dev` (opens on `http://localhost:5173`), `npm run build` and `npm run preview`, `npm test` and `npm run test:e2e`, where the Whisper model downloads from and that it is cached by the browser after the first run, and a short list of keyboard shortcuts.
- You cannot reach the user's machine. Assume no GPU and possibly no network access to `huggingface.co` in your environment. Section 9 says what to test instead.

## 4. Features

### 4.1 Input
- A "Open video" button and drag-and-drop onto the window. Accept `video/*`. Read metadata with mediabunny `Input` (duration, width, height, frame rate, video codec, audio codec, rotation). Show the file name and these facts as pills in the top bar.
- If the video codec cannot be decoded by this browser (`canDecodeVideo` false), show a plain error and stop.

### 4.2 Preview and transport
- A preview canvas, sized to the output aspect of the selected clip (16:9 or 9:16), letterboxed inside the available area. Draw every frame with the shared renderer from section 6: the current video frame with the clip's fit, then any active caption.
- Drive drawing from `video.requestVideoFrameCallback` when available, else `requestAnimationFrame`.
- Timecode display in `HH:MM:SS.mmm` with `font-variant-numeric: tabular-nums`.
- When a clip is selected, playback is confined to that clip and loops at its out point.

### 4.3 Timeline and clips
- One source, many clips. A clip is `{ id, inSec, outSec, aspect: '16:9' | '9:16', panX: number (0..1, default 0.5), captions: Caption[], transcriptStatus }`.
- Timeline is a canvas strip under the preview: full duration mapped to width, zoomable (keys below) and scrollable, with the playhead, the pending in and out markers, and each clip drawn as a rounded pill labelled with its index. Click to seek. Drag a pill edge to trim; dragging snaps to frames.
- Clip list panel on the left: index, in, out, duration, aspect toggle, transcript status pill (none, running with a percent, done, failed), delete.
- The pending in/out markers become a clip on Enter. A clip cannot be shorter than 0.5 s or longer than `MAX_CLIP_SECONDS`; show a warning pill instead of creating it.

### 4.4 Keyboard first
All controls work from the keyboard with the focus on the app shell. Shortcuts are listed in a popover opened with `?`.

| Key | Action |
|---|---|
| Space | Play or pause |
| J, K, L | Reverse (seek back 1 s repeatedly), pause, play; repeated L doubles rate up to 4x |
| Left, Right | Step one frame (1 / frameRate) |
| Shift+Left, Shift+Right | Step 1 s |
| Home, End | Start or end of source, or of selected clip |
| I, O | Set in or out at playhead |
| Enter | Create clip from in/out |
| Up, Down | Select previous or next clip |
| [ , ] | Jump to selected clip in or out |
| Backspace or Delete | Delete selected clip (confirm snackbar with Undo) |
| A | Toggle selected clip aspect 16:9 and 9:16 |
| , and . | Pan selected 9:16 clip left or right by 2 percent |
| T | Transcribe selected clip |
| E | Export selected clip |
| Shift+E | Export all clips, one after another |
| - and = | Zoom timeline out and in |
| Ctrl or Cmd+Z, Shift+Ctrl or Cmd+Z | Undo, redo |
| Esc | Clear in/out markers or close popover |

### 4.5 Caption editing
- Right panel lists the selected clip's captions: start, end, text. Rows are dense; the active caption (containing the playhead) is highlighted and the list scrolls to it.
- Click a row to seek to its start. Click the text to edit inline; Enter commits, Esc cancels. Editing text keeps start and end.
- Timecode fields are editable; Alt+Left and Alt+Right on a focused field nudge by 10 ms, Shift adds 100 ms. End cannot precede start plus 100 ms; neighbours are not pushed.
- Row actions as icon buttons: split at text cursor (time divided proportionally to character count), merge with next, delete. Add caption button inserts a 1 s caption at the playhead if that range is free.
- Re-running transcription asks to replace existing captions via a snackbar with Replace and Cancel.

### 4.6 Export
- Export the selected clip or all clips. Progress bar with percent from mediabunny `onProgress` and a cancel button. Output filename `<source basename>_clip<index>_<16x9|9x16>.mp4`.
- Output size: 16:9 is 1920x1080, 9:16 is 1080x1920; if the source is smaller than the target on the fitted axis, scale down to the source's size instead of upscaling, keeping even dimensions. Frame rate equals source. 16:9 from a 16:9 source is a plain scale; 9:16 is a `cover` crop of the source centred at `panX`. A 16:9 clip from a portrait source uses `contain` with black bars.
- Audio is included per the codec policy in section 2. Audio and video must stay in sync: both tracks come from the same `Conversion` with the same `trim`, which guarantees a common timebase.

## 5. The caption style, exact values

This is the only style. Values come from a decoded Premiere preset. Do not show the preset's name anywhere in the product. Put all constants in `src/captions/style.ts`.

| Property | Value | Status |
|---|---|---|
| Font family | Dosis, self-hosted from `@fontsource/dosis` (OFL 1.1), subsets latin and latin-ext | Confirmed |
| Weight | 700 | Confirmed |
| Font size | 115 px when the frame's shorter side is 1080 px, so `fontPx = 115 * min(w, h) / 1080` | Size 115 confirmed; reference frame is an **Assumption** |
| Fill colour | `#ffffff` | **Assumption**, not in decoded data |
| Letter spacing | `CAPTION_TRACKING_EM = 0` | **Assumption**; the preset holds an unexplained value of 15 that may be tracking or leading. Tune by eye and report what you chose |
| Horizontal anchor | Text centred at `x = 0.5 * w` | Confirmed |
| Vertical anchor | Text box vertical centre at `y = CAPTION_ANCHOR_Y * h`, `CAPTION_ANCHOR_Y = 0.885` | Decoded layers compound to 0.87 to 0.90; single constant, tune within that range |
| Shadow 1 | Black, opacity 100 percent, offset 0, softness 45 | Confirmed |
| Shadow 2 | Black, opacity 50 percent, offset 0, softness 45 | Confirmed |
| Shadow softness mapping | `shadowBlur = 45 * min(w, h) / 1080` canvas px | **Assumption**: Premiere softness is not the same unit as canvas blur. Tune by eye so the glow extends roughly one third of a cap height beyond the glyph edge and report |
| Entrance animation | Scale only: 0.95 to 1.00, starts 10 ms after caption start, lasts 57 ms, ease-in-out | Confirmed |
| Easing curve | Smoothstep `3t^2 - 2t^3` | **Assumption** standing in for Premiere's default eased bezier handles |
| Scale origin | Centre of the text's bounding box at the anchor | **Assumption** |
| Lines | One line, never wraps | Decision |
| Other animation | None: no fade, no slide, no exit, no per-word effects | Confirmed |

Rendering recipe, in order, for a caption at local time `t = now - caption.start`:

1. `s = t < 0.010 ? 0.95 : t > 0.067 ? 1 : 0.95 + 0.05 * smoothstep((t - 0.010) / 0.057)`.
2. `ctx.save(); ctx.translate(0.5 * w, CAPTION_ANCHOR_Y * h); ctx.scale(s, s);`
3. `ctx.font = '700 ' + fontPx + 'px Dosis'; ctx.textAlign = 'center'; ctx.textBaseline = 'middle'; ctx.letterSpacing = CAPTION_TRACKING_EM + 'em'`.
4. Pass A: `ctx.shadowColor = 'rgba(0,0,0,1)'; ctx.shadowBlur = blur; ctx.shadowOffsetX = ctx.shadowOffsetY = 0; ctx.fillStyle = '#fff'; ctx.fillText(text, 0, 0)`.
5. Pass B: same with `shadowColor = 'rgba(0,0,0,0.5)'`, `fillText` again.
6. Pass C: `ctx.shadowColor = 'transparent'; ctx.fillText(text, 0, 0)` for a crisp fill. `ctx.restore()`.

Fonts must be loaded before any render: `await document.fonts.load('700 115px Dosis')` once at startup and again before export; export must not start until it resolves.

## 6. Shared renderer

`src/captions/render.ts` exports one pure function:

```ts
export type FrameLayout = { w: number; h: number; fit: 'contain' | 'cover'; panX: number };
export function renderFrame(
  ctx: CanvasRenderingContext2D | OffscreenCanvasRenderingContext2D,
  drawSource: (ctx) => void,   // draws the video frame already fitted into w x h
  captions: Caption[],         // the clip's captions, times in clip-local seconds
  tLocal: number,              // clip-local time in seconds
  layout: FrameLayout,
): void
```

It clears to black, calls `drawSource`, finds the caption with `start <= tLocal < end`, and draws it with the recipe in section 5. It has no access to the DOM, timers or state, so a given `(captions, tLocal, layout)` always produces the same pixels.

- Preview calls it with `drawSource = ctx => drawFitted(ctx, videoElement, layout)` and `tLocal = video.currentTime - clip.inSec`.
- Export calls it from mediabunny's `video.process` callback with `drawSource = ctx => sample.drawWithFit(ctx, { fit, crop })` and `tLocal = sample.timestamp`. Confirmed in a spike: inside `Conversion` with `trim`, `process` receives samples whose `timestamp` is already relative to `trim.start`, so no offset is needed. Assert this in a test anyway.

`drawFitted` for the preview must do the same maths as `drawWithFit` plus `crop`: compute the source rectangle for `cover` centred at `panX`, or the destination rectangle for `contain`. Put the maths in `src/captions/fit.ts` and unit test it.

Export pipeline sketch, main thread (fonts are unavailable in workers):

```ts
const input = new Input({ source: new BlobSource(file), formats: ALL_FORMATS });
const canvas = new OffscreenCanvas(outW, outH); const ctx = canvas.getContext('2d')!;
const output = new Output({ format: new Mp4OutputFormat({ fastStart: 'in-memory' }), target: new BufferTarget() });
const conversion = await Conversion.init({
  input, output,
  trim: { start: clip.inSec, end: clip.outSec },
  video: {
    codec: 'avc', bitrate: QUALITY_HIGH, processedWidth: outW, processedHeight: outH,
    process: (sample) => { renderFrame(ctx, c => sample.drawWithFit(c, { fit, crop }), clip.captions, sample.timestamp, layout); return canvas; },
  },
  audio: audioPlan, // {} to let mediabunny copy, { codec: 'aac' } or { codec: 'opus' } per section 2
});
conversion.onProgress = p => setProgress(p);
await conversion.execute();
const blob = new Blob([output.target.buffer!], { type: 'video/mp4' });
```

Decide `audioPlan` before export using `canEncodeAudio('aac')` and the source audio codec, and inspect `conversion.discardedTracks` after `init`; if the audio track was discarded, re-init with the next fallback rather than exporting silent video.

## 7. Transcription

- Worker file `src/transcribe/worker.ts`. Main-thread client `src/transcribe/client.ts` posts `{ type: 'transcribe', audio: Float32Array (16 kHz mono), language }` and receives progress and `{ words: { text, start, end }[] }`.
- Audio extraction, in `src/media/audio-extract.ts`: open the source with mediabunny, `AudioSampleSink.samples(clip.inSec, clip.outSec)`, convert each `AudioSample` to float planar, downmix to mono, resample to 16 kHz with linear interpolation, and concatenate. Never call `decodeAudioData` on the whole file.
- Pipeline: `pipeline('automatic-speech-recognition', 'onnx-community/whisper-base_timestamped', { device, dtype })`. Call with `{ return_timestamps: 'word', chunk_length_s: 30, stride_length_s: 5, language: 'en' }` (**Assumption**: English; keep `language` a constant in `style.ts`'s sibling `transcribe/config.ts`).
- Device policy: try `webgpu` with `dtype: { encoder_model: 'fp32', decoder_model_merged: 'q4' }` only if `navigator.gpu?.requestAdapter()` returns an adapter. Validate the result: if it throws, or word timestamps are not strictly non-decreasing, or more than 20 percent of words have zero duration, discard and rerun on `wasm` with `dtype: 'q8'`. Word timestamps on WebGPU have a history of regressions, so WASM is the trusted path and WebGPU is a speed bonus. Set `env.backends.onnx.wasm.numThreads` to `navigator.hardwareConcurrency` when `crossOriginIsolated` is true.
- Serve with `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp` in Vite dev and preview so `SharedArrayBuffer` and WASM threads work. If model fetches from `huggingface.co` are blocked by COEP, switch to `credentialless` and note it.
- The model (roughly 80 to 150 MB depending on dtype) downloads on first use and is cached by transformers.js in the browser's Cache storage. Show download progress as a percent in the clip's status pill.
- Stub: when the URL has `?stub=1` or `VITE_STUB_TRANSCRIPT=1`, the client returns `tests/fixtures/stub-transcript.json` (commit this JSON: about 40 words over 8 s, plausible speech) shifted to the clip's duration. Browser tests use the stub.
- Chunking words into captions, `src/captions/chunk.ts`, pure and unit tested. Measure with the real caption font on a scratch canvas. Pack words greedily left to right into one caption while all hold: measured text width at `fontPx` is at most `0.86 * w`, at most `MAX_WORDS_PER_CAPTION = 4` words, the gap to the next word is under `0.6 s`, and the previous word does not end with `. ! ?`. Caption start is its first word's start; end is its last word's end, extended to the next caption's start when that gap is under 0.5 s, and never shorter than 0.4 s. All three thresholds are **Assumptions**; keep them as constants.
- Captions store times relative to the clip's in point. Trimming a clip's in point shifts them; trimming out point drops captions that start past the new end.

## 8. Interface rules (the editor's chrome only, never the captions)

- Dark only. Dense, information-first layout: top bar with file facts and actions, clip list left, preview centre with the timeline beneath, caption list right.
- Colours: page `#0a0e16`, surface `#151a24`, raised chrome `#1e2430`, pressed or hovered chrome `#2a313e`. Text in exactly three levels: `#ffffff`, `#a8afbd`, `#858c9a`; never dimmer than `#858c9a`. Dividers and borders always `1px solid rgba(92,104,128,.42)`. Control fills `rgba(118,130,156,.20)` at rest and `rgba(118,130,156,.32)` on hover. Accent `#0a84ff` only on actionable things and focus rings. Semantic colours only for state: good `#30d158`, destructive `#ff453a`, warn `#ff9f0a`.
- Shape: content containers are sharp, 0 radius, hairline border, no shadow. Pressable things are rounded 10 px, 14 px on floating surfaces. Floating shadow for popovers and snackbars only: `0 14px 44px rgba(0,0,0,.4)`.
- Type: `-apple-system, BlinkMacSystemFont, system-ui, "Segoe UI", sans-serif`; do not name "SF Pro Text". Monospace: `"JetBrains Mono", ui-monospace, SFMono-Regular, Menlo, monospace`. Tracking `-0.026em` at 30 px and up, `-0.018em` around 17.5 px, `-0.012em` around 15 px. Section labels uppercase 12 px, weight 700, `+0.06em`, muted colour. Any digits in a column, including every timecode, use `tabular-nums`.
- Motion: easing `cubic-bezier(.32,.72,0,1)`; row hover transition 140 ms.
- Copy: no explanatory subtitles, filler notes or onboarding text. Never use the middle dot character as a separator anywhere in UI text; use icons, pills or line breaks instead.
- Put every token in `src/styles/tokens.css` as custom properties and use only those.

## 9. Verification

### 9.1 Unit tests (Vitest, jsdom not needed for pure modules)
- `fit.ts`: cover and contain rectangles for 16:9 to 9:16 with `panX` 0, 0.5, 1; contain for portrait into 16:9.
- `chunk.ts`: packing respects width, word cap, gap and punctuation; end-extension and minimum duration; a stubbed `measureText`.
- Caption scale function: values at `t = 0, 0.010, 0.0385, 0.067, 1.0` are `0.95, 0.95, 0.975, 1.0, 1.0` within `1e-6`.
- Store: clip creation limits, caption split and merge, undo and redo, in-point trim shifting captions.
- Export planner: audio plan selection for the combinations source AAC with and without AAC encoder, source Opus, no audio track.

### 9.2 Browser tests (Playwright, headless Chromium, `--trace off`)
Headless Chromium in the cloud has no GPU adapter and no AAC encoder but does have software H.264 encode and decode, Opus encode, AAC decode, `OffscreenCanvas`, and `SharedArrayBuffer` under COOP and COEP. It must be reached over `http://localhost`. All of this was measured on Chromium 153 in this environment; re-run the probe below in yours and paste its output in the PR.

Probe, run once in `page.evaluate` on the app origin and print as JSON: `isSecureContext`, `crossOriginIsolated`, `!!navigator.gpu && !!(await navigator.gpu.requestAdapter())`, `VideoEncoder.isConfigSupported({ codec: 'avc1.640028', width: 1920, height: 1080, bitrate: 8e6, framerate: 30 })`, `AudioEncoder.isConfigSupported({ codec: 'mp4a.40.2', sampleRate: 48000, numberOfChannels: 2, bitrate: 128000 })`, same for `opus`, `AudioDecoder.isConfigSupported({ codec: 'mp4a.40.2', ... })`.

Fixture: `scripts/make-fixture.sh` generates `tests/fixtures/test.mp4` with ffmpeg (present in the cloud image; if missing, `apt-get install -y ffmpeg`): `ffmpeg -y -f lavfi -i testsrc2=size=1280x720:rate=30:duration=12 -f lavfi -i sine=frequency=440:duration=12 -c:v libx264 -pix_fmt yuv420p -c:a aac -shortest tests/fixtures/test.mp4`. The file is gitignored; the e2e setup regenerates it.

Tests:
1. Load: set the file input to the fixture, assert duration pill reads `00:00:12.000` and codec pills read `H.264` and `AAC`.
2. Clip: seek to 2 s, press I, seek to 7 s, press O, Enter; assert one clip row with in `00:00:02.000` and out `00:00:07.000`.
3. Transcribe with stub (`?stub=1`): press T, wait for status done, assert the caption list has at least 6 rows and times are inside 0 to 5 s.
4. Export 16:9 and 9:16: press E, capture the download, write it under `tests/e2e/out/`, then run `ffprobe` from Node and assert: video `h264`, dimensions `1280x720` for 16:9 (source is smaller than 1920x1080, so no upscale) and `720x1280` for 9:16, duration within 0.1 s of 5.0, an audio stream exists with codec `aac` (copied), audio duration within 0.1 s of video duration.
5. Burn-in check: extract a frame from the export at 2.5 s with ffmpeg and assert there are near-white pixels (all channels above 235) inside the band from `0.84h` to `0.93h` and none in the band from `0.10h` to `0.40h`, where a caption is active at that time in the stub.
6. Preview equals export: a dev-only route `/#/render-test` draws `renderFrame` onto a 1080x1920 canvas with a mid-grey source and the fixed caption "never gonna give" at `tLocal = 0.5`. Screenshot it at `deviceScaleFactor: 1` and compare to the committed golden `tests/golden/caption-9x16.png` with a 0.5 percent pixel tolerance. Produce the golden in your run and commit it; that is the one PNG allowed in the repo. Also render the same caption through the export path to a single-frame MP4 and compare the decoded frame against the golden with a tolerance of 2 percent to allow for H.264 compression.
7. Pop timing: render at `tLocal = 0.0` and `tLocal = 0.5` through the render-test route and assert the measured text width at 0.0 is 0.95 times the width at 0.5, within 1 percent, using `getImageData` to find the leftmost and rightmost near-white columns.

### 9.3 Caption style checklist
Verify each with the render-test route and record the result in the PR:
- Font resolves to Dosis 700, not a fallback: `document.fonts.check('700 115px Dosis')` is true before drawing.
- `fontPx` is 115 on a 1920x1080 frame and 115 on a 1080x1920 frame.
- Text horizontal centre is within 2 px of `w / 2`; vertical centre of the glyph band is within 1 percent of `h` from `0.885 * h`.
- Shadow is a soft glow with no offset: the dark halo is symmetric left and right of the text within 2 px.
- Two shadow passes at 100 percent and 50 percent: the halo's darkest ring is darker than a single-pass render (compare against a debug single-pass render in the same test, alpha difference at least 0.1 at the glyph edge).
- Scale is 0.95 at caption start, 1.0 at 67 ms and after, nothing else animates: no alpha change, no position change between `tLocal` 0.0 and 0.5 other than width.
- Captions never wrap: a 40-character input string must be split by the chunker, not wrapped by the renderer.

### 9.4 What cannot be verified in the cloud, and must be listed in the PR
- Real Whisper inference in the browser: likely blocked by network policy or too slow without a GPU. If `huggingface.co` is reachable, run the smallest possible real transcription of the 12 s fixture on `wasm` and report the time; otherwise state it was not run.
- WebGPU path for transcription.
- AAC re-encode path, hardware encoding speed, Firefox and Safari behaviour, exports over about 60 s, actual visual tuning of shadow softness and tracking by eye on real footage.

## 10. Build order and file layout

Commit after each step.

1. Scaffold: Vite, React, TypeScript strict, Zustand, Vitest, Playwright, `.gitignore`, `tokens.css`, README skeleton, COOP and COEP headers, `@fontsource/dosis` 700 imported in `main.tsx`.
2. Media open and preview: `media/source.ts`, `<video>` with blob URL, preview canvas, transport keys, timecode.
3. Timeline and clips: store, canvas timeline, in/out, clip list, aspect toggle, pan, undo.
4. Caption style and renderer: `style.ts`, `fit.ts`, `render.ts`, render-test route, golden image, unit tests, style checklist.
5. Export: `media/export.ts` without captions first, then with `renderFrame`; audio plan; download; progress and cancel; e2e tests 1, 2, 4.
6. Transcription: `audio-extract.ts`, worker, client, stub, chunker, status pills; e2e test 3 and 5.
7. Caption editor: list, inline edit, timecode nudge, split, merge, delete, add.
8. Persistence, shortcuts popover, snackbars, unsupported-browser notice, README complete, PR.

```
.gitignore
README.md
build-prompt.md            (this file, already present)
index.html
package.json
vite.config.ts             COOP and COEP headers for dev and preview
tsconfig.json
playwright.config.ts
vitest.config.ts
scripts/make-fixture.sh
src/main.tsx
src/app/App.tsx
src/app/keys.ts            one keymap, section 4.4
src/state/store.ts         Zustand store with undo snapshots
src/state/types.ts
src/state/persist.ts       IndexedDB autosave
src/media/source.ts        open file, mediabunny Input, metadata, decode checks
src/media/audio-extract.ts clip range to 16 kHz mono Float32Array
src/media/export.ts        Conversion pipeline and audio plan
src/captions/style.ts      every caption constant from section 5
src/captions/fit.ts
src/captions/render.ts     renderFrame
src/captions/chunk.ts
src/transcribe/config.ts
src/transcribe/worker.ts
src/transcribe/client.ts
src/transcribe/stub.ts
src/ui/TopBar.tsx
src/ui/Preview.tsx
src/ui/Timeline.tsx
src/ui/ClipList.tsx
src/ui/CaptionList.tsx
src/ui/ExportDialog.tsx
src/ui/Snackbar.tsx
src/ui/Shortcuts.tsx
src/ui/RenderTest.tsx      dev-only route
src/styles/tokens.css
src/styles/app.css
tests/unit/*.test.ts
tests/e2e/*.spec.ts
tests/e2e/probe.ts         capability probe from section 9.2
tests/fixtures/stub-transcript.json
tests/golden/caption-9x16.png
```

## 11. Acceptance criteria

The PR is complete when all of these hold and the PR says which were checked by test, by hand in headless Chromium, or not checked:

1. `npm run build` succeeds with zero TypeScript errors; `npm test` and `npm run test:e2e` pass in headless Chromium.
2. A 1280x720 12 s H.264 and AAC fixture opens, shows correct metadata, plays, scrubs and steps by frame.
3. In and out with I, O and Enter create a clip; clips list, select, trim and delete; aspect toggles; undo restores.
4. The stub transcript produces captions that obey the chunking rules; captions can be edited, split, merged, nudged and deleted.
5. Exported MP4 matches section 9.2 test 4 for both aspects, with copied AAC audio in sync.
6. The burned-in caption in the export matches the preview renderer within the tolerances in tests 5 and 6, and the style checklist in 9.3 passes.
7. No file larger than 1 MB and no media or model file is committed; `.gitignore` is present from the first commit.
8. UI uses only the tokens in section 8; no middle dot characters appear in any string under `src/`; a grep for U+00B7 returns nothing.
9. README explains local setup and shortcuts; the PR lists everything not verified.
10. Every **Assumption** from this document that you tuned is listed in the PR with the value you chose.
