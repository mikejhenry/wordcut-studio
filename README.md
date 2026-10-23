# Wordcut Studio — browser prototype

An auto-caption video editor where all the heavy lifting (speech recognition, rendering, encoding) runs on the user's device. This prototype proves the workflow and the caption engine before porting to native apps.

## Run it locally

The speech model downloads from Hugging Face on first use, so the page must be served over http (not opened as a file):

```bash
cd wordcut
python3 -m http.server 8080
# open http://localhost:8080 in Chrome, Edge, or Safari 18+
```

Use Chrome or Edge for WebGPU acceleration. Without WebGPU it falls back to the CPU (slower, still works).

## Project flow (modeled on DaVinci Resolve's pages)

| Page | What happens |
|---|---|
| **Media** | Import video/audio, pick a Whisper model, generate captions on-device |
| **Edit** | Timeline + caption list. Fix text, nudge in/out points, split (S), merge, delete |
| **Captions** | Style presets (Punch, Karaoke, Boxed, Clean, Terminal, Story, Comic, Neon) plus font, color, outline, glow, animation and layout controls |
| **Deliver** | 9:16 / 4:5 / 1:1 / 16:9 / source, crop or blurred-fit, 720p/1080p, 24/30/60 fps. Render burned-in MP4/WebM, or export SRT/VTT |

Keyboard: Space play/pause, S split at playhead, ←/→ step one frame, Delete removes the selected caption.

## Architecture

All in `index.html` for now, organized as modules that map 1:1 onto the native rewrite:

1. **Project model** — `{ name, media, captions[{ s, e, words[{ t, s, e }] }], style, out }`. Saved as `.wordcut.json`. Media is referenced by name and relinked on open (like Resolve).
2. **Caption engine** — groups timed words into captions (max words, pauses > 0.6 s, sentence ends, 3.2 s cap). Re-splitting works from the word list, so text edits survive style changes. SRT/VTT export.
3. **Style presets** — plain data objects. Easy to ship more as paid packs.
4. **Renderer** — one `drawFrame(ctx, W, H, t)` used for both preview and export, so what you see is what you get. Per-word states (upcoming / active / spoken) drive the animations.
5. **Transcriber** — Whisper via transformers.js in a Web Worker, WebGPU with CPU fallback, word-level timestamps (falls back to estimated word timing if a model can't give them).
6. **UI** — pages, viewer, timeline, inspector.
7. **Deliver** — canvas + MediaRecorder. Real-time only; this is the main prototype limitation.

## Known limits of the web prototype

- Export runs in real time and needs the tab visible. Next step: WebCodecs `VideoEncoder` + an MP4 muxer for faster-than-real-time export.
- Very long videos (30+ min) can run out of browser memory during transcription.
- Inside the claude.ai preview frame, model downloads and file saves are blocked. Use the demo clip there; run locally for the real thing.

## Native port plan

| Module | iOS / macOS | Android | Windows / Linux |
|---|---|---|---|
| Transcriber | whisper.cpp + Core ML, or Apple SpeechAnalyzer | whisper.cpp (NEON/Vulkan) | whisper.cpp (CUDA / Vulkan / CPU) |
| Renderer | Metal or Skia | Skia / OpenGL ES | Skia |
| Encoder | AVFoundation / VideoToolbox | MediaCodec | Media Foundation / NVENC / VA-API |
| UI | Flutter (shared) or SwiftUI | Flutter | Tauri or Flutter desktop |

Keep modules 1–4 in a shared C++ (or Rust) core so captions look identical on every platform.

## Next milestones

1. Drag to resize/move caption blocks on the timeline
2. Speaker detection and per-speaker colors
3. Emoji/keyword highlighting (auto-color words like numbers and names)
4. WebCodecs fast export
5. Watermark + ad slot for the free tier; premium preset packs
