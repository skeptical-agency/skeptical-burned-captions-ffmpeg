# skeptical-burned-captions-ffmpeg (Grok skill)

TikTok and Shorts style word-timed captions burned into vertical video with whisper timings, your script text, PIL-rendered PNGs and ffmpeg overlay. No CapCut, no libass.

## Install

This repo is the skill folder: `SKILL.md` sits at the top level. Clone it into a skills directory.

User scope (all your projects), macOS or Linux:

    mkdir -p ~/.grok/skills
    git clone https://github.com/skeptical-agency/skeptical-burned-captions-ffmpeg.git ~/.grok/skills/skeptical-burned-captions-ffmpeg

User scope, Windows PowerShell (assumes Grok reads skills from `$HOME\.grok\skills`):

    New-Item -ItemType Directory -Force "$HOME\.grok\skills"
    git clone https://github.com/skeptical-agency/skeptical-burned-captions-ffmpeg.git "$HOME\.grok\skills\skeptical-burned-captions-ffmpeg"

Project scope (shared via git):

    mkdir -p .grok/skills
    git clone https://github.com/skeptical-agency/skeptical-burned-captions-ffmpeg.git .grok/skills/skeptical-burned-captions-ffmpeg

No git? Download the repo as a zip from GitHub and unzip it into a folder named `skeptical-burned-captions-ffmpeg` inside the skills directory.

More skills: [skeptical-agency/grok-skills](https://github.com/skeptical-agency/grok-skills).

## Requirements

- `ffmpeg` and `ffprobe` on your PATH (macOS: `brew install ffmpeg`; Linux: `sudo apt install ffmpeg`; Windows: `winget install Gyan.FFmpeg`)
- Python 3 with `faster-whisper` and `Pillow` (`python3 -m pip install faster-whisper Pillow`)

## Use

Ask Grok things like:

> burn word-timed captions into this 9:16 video from the terminal, using my script for the text: <video> <script>

SKILL.md has a plain-English intro, a worked example, the exact commands and the known pitfalls.

## License

MIT, see LICENSE.
