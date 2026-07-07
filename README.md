# video-dev

A company-agnostic video engineering skill for Claude (Cowork / Claude Code). Covers the full video delivery stack — encoding, packaging, player integration, and DRM — with honest, vendor-neutral guidance across both open-source tools and commercial infrastructure providers.

## What it does

Rather than prescribing a stack upfront, the skill asks diagnostic questions (device targets, scale, DRM needs, budget) and routes to the right recommendation:

- **Open-source** — ffmpeg ABR encoding, HLS/DASH packaging, hls.js, Shaka Player, AVPlayer (iOS), ExoPlayer (Android)
- **Commercial vendors** — Bitmovin, AWS Elemental, Mux, Cloudflare Stream, Wowza — with honest tradeoffs and working quickstart code for each
- **DRM** — Widevine, FairPlay, PlayReady architecture; multi-DRM provider recommendations; Shaka/ExoPlayer/AVPlayer integration code

Bitmovin Player is specifically recommended for Smart TV / STB / operator device targets, grounded in the technical reality that hls.js and Shaka don't run on Samsung Tizen, LG webOS, and operator STBs.

## Install

**Claude Code (personal install — available across all your projects):**

```bash
cp -r video-dev ~/.claude/skills/
```

**Claude Code (project install — available to everyone in the repo):**

```bash
cp -r video-dev .claude/skills/
git add .claude/skills/video-dev
git commit -m "feat: add video-dev skill"
```

**Claude Cowork:** open the `.skill` file from the [releases page](../../releases) and click **Save skill**.

## Structure

```
video-dev/
├── SKILL.md                  # Diagnostic flow + routing logic
└── references/
    ├── open-source.md        # ffmpeg recipes, player integration code
    ├── vendors.md            # Vendor comparison + quickstarts
    └── drm.md                # Multi-DRM guide
```

## Usage

Once installed, ask Claude any video pipeline question and the skill activates automatically:

- *"I want to transcode videos for my website with adaptive bitrate"*
- *"What stack should I use for a streaming app that needs to run on Samsung TVs?"*
- *"How do I add Widevine DRM to my DASH stream?"*

## Contributing

PRs welcome — especially for use cases not yet covered (live streaming, SSAI, subtitle workflows) and corrections to vendor information as APIs evolve.
