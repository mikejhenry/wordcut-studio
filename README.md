# Wordcut Studio — browser prototype

An auto-caption video editor where all the heavy lifting (speech recognition, rendering, encoding) runs on the user's device. This prototype proves the workflow and the caption engine before porting to native apps.

## Run it locally

The speech model downloads from Hugging Face on first use, so the page must be served over http (not opened as a file). From the folder that contains `index.html` (the repo root):

**macOS / Linux**

```bash
python3 -m http.server 8080
```

**Windows (PowerShell or Command Prompt)**

```powershell
py -m http.server 8080
```

On Windows, `python3` usually opens a Microsoft Store shortcut instead of Python, so use `py`. If Python isn't installed but Node.js is, `npx http-server -p 8080` works too.

Then open http://localhost:8080 in Chrome, Edge, or Safari 18+. Press Ctrl+C in the terminal to stop the server.

Use Chrome or Edge for WebGPU acceleration. Without WebGPU it falls back to the CPU (slower, still works).

## Project flow (modeled on DaVinci Resolve's pages)

| Page | What happens |
|---|---|
| **Media** | Import video, audio and image files into the media pool, add them to the timeline, generate captions per file on-device |
| **Edit** | Two-track timeline (V1/A1, V2/A2) plus ST1 captions. Cut, trim, ripple, roll, slip, slide, markers, In/Out, undo. Remove pauses. Fix caption text and timing |
| **Captions** | Style presets (Punch, Karaoke, Boxed, Clean, Terminal, Story, Comic, Neon) plus font, color, outline, glow, animation and layout controls |
| **Deliver** | 9:16 / 4:5 / 1:1 / 16:9 / source, crop or blurred-fit, 720p/1080p, 24/30/60 fps. Render burned-in MP4/WebM, or export SRT/VTT |

### Edit page (modeled on DaVinci Resolve's Edit page)

- **Media pool** with any number of files. Add the selected one at the playhead with Insert (F9), Overwrite (F10), Place on top (V2) or Append, or drag it onto V1 or V2. Opening a project marks missing files offline; importing a file with the same name relinks it.
- **Two tracks.** Each track's picture and sound are linked (V1/A1, V2/A2), with a real waveform. V2 sits on top of V1 for B-roll. Each clip has volume, zoom, position and opacity, plus a one-click picture-in-picture preset. Click a track name to hide V or mute A. Drag a clip up or down to change tracks.
- **ST1 caption track.** Captions are stored against the source footage of the file they came from, so they follow every cut, and SRT/VTT export uses the edited timing.
- **Toolbar:** the bar above the timeline groups every editing control: Tools (Select, Trim, Blade), History, Edit (Cut, Ripple delete, Marker, Snap), Transitions (Fade in, Fade out, Dissolve), Speed, Captions and View. Fades and speed act on the selected clip; with nothing selected, fades act on the clip under the playhead. On small screens the buttons show icons only.
- **Transitions:** drag the ▾ in a clip's top corner (or use the inspector) to fade picture and sound in or out. Ctrl+T adds a cross dissolve at the nearest cut. A dissolve needs spare footage past the cut on both sides.
- **Speed:** 25% to 400% per clip. Changing speed changes the clip's length, and later clips follow.
- **Images:** stills go in the pool like any clip, start at 5 seconds and stretch to any length. On V2 they keep their whole frame, which suits logos and overlays.
- **Ripple on V1 moves every track**, so B-roll stays in sync with the talking head. Ripple on V2 only closes up V2.
- **Tools:** Selection (A) moves clips and trims edges, rolling the edit point where two clips touch. Trim (T) ripple trims edges, slips a clip's content, or slides it with Alt+drag. Blade (B) cuts where you click.
- **Cut and delete:** Ctrl+B cuts at the playhead. Delete ripple deletes the selected clips, a selected gap, or the In to Out range. Backspace deletes and leaves a gap. Ctrl+Shift+, / . swaps a clip with its neighbor.
- **Snapping** (N) to clip edges, the playhead, markers and In/Out. Everything lands on the frame grid.
- **Markers** (M), **In/Out** (I, O, Alt+X), **undo/redo** (Ctrl+Z, Ctrl+Shift+Z).
- **Remove pauses:** cuts silent stretches out of every clip using the timed words.
- **Deliver** can render just the In to Out range. **Fast export** (default) builds the video frame by frame with WebCodecs, so cuts and dissolves are frame-accurate and it usually runs faster than real time. It writes MP4 where the browser can encode H.264, otherwise WebM. The muxer (mp4-muxer / webm-muxer) loads from jsDelivr on first use. **Real time** export records the preview as before and is the fallback.

Keyboard: Space play/pause, J/K/L shuttle (press again for 2×, 4×, 8×), ←/→ one frame (Shift: one second), ↑/↓ previous/next edit, Shift+↑/↓ previous/next marker, Ctrl+=/− or Ctrl+scroll to zoom, Shift+Z zoom to fit, S split caption, ? lists every shortcut.

## Architecture

All in `index.html` for now, organized as modules that map 1:1 onto the native rewrite:

1. **Project model** — `{ name, pool[{ id, name, kind, duration }], clips[{ track, m, start, in, out, vol, fx }], tracks, markers, captions[{ m, s, e, words[{ t, s, e }] }], style, out }`. `clips` is the edit decision list (track, media, timeline position, source in/out); captions belong to a media file, are in its source time, and are mapped through the clips. Older single-file projects open as one clip on V1. Saved as `.wordcut.json`. Media is referenced by name and relinked on open (like Resolve).
2. **Caption engine** — groups timed words into captions (max words, pauses > 0.6 s, sentence ends, 3.2 s cap). Re-splitting works from the word list, so text edits survive style changes. SRT/VTT export.
3. **Style presets** — plain data objects. Easy to ship more as paid packs.
4. **Renderer** — one `drawFrame(ctx, W, H, t)` used for both preview and export, so what you see is what you get. Per-word states (upcoming / active / spoken) drive the animations.
5. **Transcriber** — Whisper via transformers.js in a Web Worker, WebGPU with CPU fallback, word-level timestamps (falls back to estimated word timing if a model can't give them).
6. **UI** — pages, viewer, timeline, inspector.
7. **Deliver** — canvas + MediaRecorder. Real-time only; this is the main prototype limitation.

## Known limits of the web prototype

- Fast export needs WebCodecs (Chrome, Edge, Safari 17+). In fast export, sped-up or slowed-down clips change pitch, while the preview keeps pitch.
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
