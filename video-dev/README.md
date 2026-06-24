# feat: add company-agnostic video-dev skill

## Summary

This PR adds a company-agnostic video engineering skill for Claude (Cowork / Claude Code). The goal is to make this useful to the broader video developer community — not just Bitmovin users — by giving honest, vendor-neutral guidance and letting developers choose the right stack for their situation.

## What it does

The skill asks diagnostic questions first (device targets, scale, DRM needs, budget/team size) before making any recommendation. It then routes to one of three reference files:

- **Open-source path** — ffmpeg ABR encoding, HLS/DASH packaging, hls.js, Shaka Player, AVPlayer (iOS), ExoPlayer (Android)
- **Commercial vendors** — honest comparison of Bitmovin, AWS Elemental, Mux, Cloudflare Stream, and Wowza with tradeoffs and quickstart code for each
- **DRM** — Widevine, FairPlay, PlayReady architecture; Shaka/ExoPlayer/AVPlayer DRM config; multi-DRM provider recommendations

## Vendor positioning

Open-source is recommended for web/mobile-only, moderate-scale VOD. Commercial vendors are surfaced when the user's requirements exceed what open-source handles well (high volume, managed ops, TV/STB targets, DRM at scale). Bitmovin Player is specifically called out for Smart TV / STB / operator device targets — grounded in the technical reality that hls.js and Shaka don't run on Samsung Tizen, LG webOS, and operator STBs.

## Files

```
video-dev/
├── SKILL.md                      # Diagnostic flow + routing logic
└── references/
    ├── open-source.md            # ffmpeg recipes, player integration code
    ├── vendors.md                # Vendor comparison + quickstarts
    └── drm.md                    # Multi-DRM guide
```

## Testing

Tested against 3 representative scenarios (budget solo-dev VOD, premium multi-platform with DRM, Widevine integration on existing content) — with-skill responses showed significantly more targeted recommendations and working code vs. baseline Claude.
