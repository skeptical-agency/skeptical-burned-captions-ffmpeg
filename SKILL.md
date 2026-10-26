---
name: skeptical-burned-captions-ffmpeg
description: "Burn word-timed punch captions into 9:16 video with whisper + PIL + ffmpeg, no CapCut, no libass. Use when you need TikTok, Shorts or Reels style word-timed burned captions from the terminal."
when-to-use: "Shorts, Reels or TikToks needing word-timed burned captions from the terminal, your ffmpeg has no libass/freetype/drawtext (Homebrew builds often don't), or you need caption text from the script but timing from whisper."
---

# Burned-in captions pipeline (whisper + PIL + ffmpeg)

## What this does (plain English)

This skill teaches your AI assistant to burn punchy, word-timed captions into vertical video entirely from the terminal, with no editing app. The key idea: whisper (a speech-to-text tool) supplies only the timing, and the caption words come from your original script, so whisper mishearings never get printed on screen. Each caption group is drawn as a PNG image with PIL (the Python imaging library), and ffmpeg overlays each PNG for exactly the right time window. It does not need `libass`, which many ffmpeg builds (including Homebrew's) do not include. The skill also lists every timing and layout bug hit in production, such as number fragments splitting, captions stacking on boundary frames, and text running off the edge of the canvas.

## Install what you need

You need `ffmpeg` and `ffprobe`, plus Python 3 (from python.org or your system package manager) with the packages the recipe uses.

macOS:

```
brew install ffmpeg
```

Linux (Debian/Ubuntu shown; on Fedora use `sudo dnf install ffmpeg`, which may need the RPM Fusion repo enabled):

```
sudo apt install ffmpeg
```

Windows (PowerShell or Command Prompt; open a NEW terminal window after installing so the tools show up on PATH):

```
winget install Gyan.FFmpeg
```

Check it worked: `ffmpeg -version` and `ffprobe -version` should both print a version.

Python packages (`faster-whisper` provides `WhisperModel`; `Pillow` provides PIL):

```
python3 -m pip install faster-whisper Pillow
```

## Worked example

A short outline of captioning one 9:16 clip, using only commands and values from the recipe below:

1. Get word timings from your audio with whisper. The small model is usually enough:

```
WhisperModel("small").transcribe(audio, word_timestamps=True)
```

2. Merge number fragments before grouping (whisper splits "$2,499" into "$2" + ",499"): merge any token matching `^[,.]\d` onto the previous token. When parsing dumped words, use `\[(s)-(e)\]\s*(\S+)`.
3. Align the script tokens onto whisper's timings (difflib SequenceMatcher), group into 1-4 word punch groups, and render each group as a PNG (Bebas Neue ~96pt, uppercase, canvas width = video width, height ~150).
4. Assert the margins at render time: ink inside the canvas with >= 40px margin on all four edges.
5. Overlay one PNG per group at y=1540 with an enable window per group (this is the exact filter shape from the recipe, with S1/E1 and S2/E2 filled in from your timings):

```
ffmpeg -i video.mp4 -i cap_001.png -i cap_002.png ... -filter_complex
"[0:v][1:v]overlay=x=(W-w)/2:y=1540:enable='between(t,S1,E1)'[v1];
 [v1][2:v]overlay=x=(W-w)/2:y=1540:enable='between(t,S2,E2)'[v2];..."
```

6. Verify by cropping the caption band with `crop=1080:260:0:1500` and scaling it up 2x, then diff the silent render against the captioned render in y 1400-1700 at 10+ timestamps.

---

The full production recipe follows.


## When to Use
- Shorts/Reels/TikToks needing word-timed burned captions from the terminal
- Your ffmpeg has no libass/freetype/drawtext (Homebrew builds often don't)
- You need caption TEXT from the script but TIMING from whisper

## Core principle
Caption TEXT must come from the original script (difflib SequenceMatcher
alignment of script tokens onto whisper timings). Whisper only supplies timing.
Whisper garbles words ("now" -> "nails") and those errors print verbatim if you
caption from the transcript.

## Pipeline

1. Transcribe with word timestamps:

   ```
   WhisperModel("small").transcribe(audio, word_timestamps=True)
   ```
   (small is usually enough; use medium on suspect windows; treat whisper text
   as phonetic, not literal.)

2. Merge number fragments BEFORE grouping: whisper splits "$2,499" into "$2" +
   ",499". Merge any token matching `^[,.]\d` onto the previous token. Note every
   word except a segment's first carries a LEADING SPACE when dumped to text;
   parse with `\[(s)-(e)\]\s*(\S+)` not `\[(s)-(e)\](\S+)` or you silently drop
   nearly every word.

3. Group into 1-4 word punch groups aligned to phrase boundaries (never orphan
   a preposition/article at the end of a card; push trailing function words to
   the next group; merge product noun phrases like "M5 PRO MACBOOK PRO" into
   atomic units before grouping).

4. Render each group as a PNG with PIL: Bebas Neue ~96pt, uppercase, white with
   black outline + drop shadow, key terms (products, prices, specs) in one
   accent color per video. Canvas = video width (e.g. 1080), PNG height ~150.

5. MARGIN ASSERT at render time (deterministic; pixel-scanning the final video
   is unreliable, bright backgrounds false-positive):
   - text ink (alpha>30) fully inside canvas with >= 40px margin on ALL FOUR edges
   - shrink font to fit MAXW (e.g. 880 of 1080); split to 2 lines if still too
     wide at the font floor; a hard-floor font that can't fit must RAISE, never
     silently overflow
   - the classic silent bug: overlay centers at x=(W-w)/2, so a group wider than
     the canvas gets a NEGATIVE x and ink runs off both edges. Assert the FINAL
     on-canvas position after centering.

6. Overlay with ffmpeg, one input PNG per group:

   ```
   ffmpeg -i video.mp4 -i cap_001.png -i cap_002.png ... -filter_complex
   "[0:v][1:v]overlay=x=(W-w)/2:y=1540:enable='between(t,S1,E1)'[v1];
    [v1][2:v]overlay=x=(W-w)/2:y=1540:enable='between(t,S2,E2)'[v2];..."
   ```
   -libass is NOT needed. Dozens of -i inputs are fine. All -i flags come
   BEFORE -filter_complex/-map (appending inputs after map options errors).

7. Placement: centered, y ~1540 of 1920 (clear of platform UI). NEVER place any
   other baked text (cards, labels) in y1500-1700 - it collides with the
   caption band. Cards go at cy = H-520 or higher (bottom ~y1478).

## Timing bugs (all hit in production)

- ENABLE IS INCLUSIVE ON BOTH ENDS. Contiguous groups (end == next start, common
  when whisper has no gap) BOTH render on boundary frames -> stacked captions.
  Cap each group: e = min(group_end + 0.15, next_start - one_frame). At 30fps
  the -0.034s margin is invisible; a stacked frame is not.
- STILL-SEGMENT -t ROUNDING DRIFTS: -loop 1 -t <dur> rounds each segment UP
  ~1 frame and the error CUMULATES across a concat (15 segments drifted pops
  0.15s past their spoken words). Plan cut points as INTEGER FRAMES (snap word
  times to n/fps), render stills with -frames:v round(dur*FPS), assert
  per-segment duration <0.05s off plan.
- QA pops by pixel-counting the animated region at pop-time +/-0.05s. A 1-frame
  late pop reads fine; 4-5 frames late fails.
- Caption "blanks" at 0.25s sampling are usually comma/breath gaps between
  groups (normal); map actual enable windows before chasing them.

## Verification
- Diff the silent render vs captioned render in the caption band
  (y 1400-1700) at 10+ timestamps spanning EVERY segment type; max pixel diff
  > ~60 = captions rendered. Captions can SILENTLY vanish after a concat
  boundary when segments carry different color metadata (see the `skeptical-ffmpeg-concat-traps` skill).
- Vision models misread small caption text on full 1080x1920 frames; crop the
  band (`crop=1080:260:0:1500`) and 2x-scale, or verify the PNGs directly.
- Accent color check must be CASE-INSENSITIVE (compare word.upper() against the
  uppercase key set) or accent silently skips already-uppercase words.
- Thumbnail safe zone: keep hook/title text within y=400..1500 of 1920 (grid
  thumbnails crop to a middle band).

## Fonts
Bebas Neue: fetch raw from google/fonts GitHub (`ofl/bebasneue/BebasNeue-Regular.ttf`);
the fonts.google.com download endpoint returns a non-zip. Pass the file path to
PIL directly, don't install system fonts.
