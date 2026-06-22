# DRM & Content Protection Reference

## DRM Systems Overview

| System | Platform | Key Server |
|--------|----------|------------|
| **Widevine** | Chrome, Firefox, Android, Chromecast | Google (L1/L2/L3 security levels) |
| **FairPlay** | Safari, iOS, tvOS, macOS | Apple (requires Apple developer program) |
| **PlayReady** | Edge, Windows, Xbox, some Smart TVs | Microsoft |

For broad coverage, implement **multi-DRM**: all three systems, served by a single packager
and license server. CENC (Common Encryption) lets you encrypt once and serve licenses per-system.

---

## Architecture Overview

```
Source Video
    │
    ▼
Packager (encrypts segments, embeds PSSH boxes)
    │
    ├── DASH manifest (.mpd) with ContentProtection elements
    └── HLS manifest (.m3u8) with #EXT-X-KEY / #EXT-X-SESSION-KEY
              │
              ▼
         CDN (serves encrypted segments — decryption happens client-side)
              │
              ▼
         Player (requests license when it encounters encrypted segment)
              │
              ▼
         License Server (validates token, returns decryption key)
```

---

## Open-Source Path

### Shaka Packager (encryption + DASH/HLS packaging)

[github.com/shaka-project/shaka-packager](https://github.com/shaka-project/shaka-packager)

```bash
# Install
docker pull gcr.io/shaka-packager/release:latest

# Package + encrypt with CENC (Widevine + PlayReady)
docker run --rm -v $(pwd):/content gcr.io/shaka-packager/release:latest \
  in=/content/source.mp4,stream=video,output=/content/video.mp4 \
  in=/content/source.mp4,stream=audio,output=/content/audio.mp4 \
  --enable_widevine_encryption \
  --key_server_url https://license.widevine.com/cenc/getcontentkey/YOUR_PROVIDER \
  --content_id $(echo -n "my-content-id" | xxd -p) \
  --signer YOUR_SIGNER \
  --aes_signing_key YOUR_AES_KEY \
  --aes_signing_iv YOUR_AES_IV \
  --mpd_output /content/manifest.mpd \
  --hls_master_playlist_output /content/master.m3u8
```

**Note:** Widevine content key server access requires a license from Google.
For development, use a test content key server or a multi-DRM provider (see below).

### Test/Dev: Clearkey (no license server needed)

Useful for integration testing without a real license server:

```bash
# Generate a random key and key ID
KEY_ID=$(openssl rand -hex 16)
KEY=$(openssl rand -hex 16)

packager \
  "in=source.mp4,stream=video,output=video.mp4" \
  "in=source.mp4,stream=audio,output=audio.mp4" \
  --enable_raw_key_encryption \
  --keys "label=:key_id=${KEY_ID}:key=${KEY}" \
  --mpd_output manifest.mpd
```

Player receives the key directly in the manifest or via a trivial JSON endpoint — no real
license server. Only use this for development; clearkey provides no meaningful protection.

---

## Multi-DRM Providers (Recommended for Production)

Running your own Widevine proxy + FairPlay license server is complex and requires direct
vendor relationships. Most teams use a multi-DRM SaaS:

| Provider | Notes |
|----------|-------|
| **Axinom DRM** | Developer-friendly, good docs, usage-based pricing |
| **EZDRM** | Long-established, broad format support |
| **BuyDRM (KeyOS)** | Enterprise, strong PlayReady support |
| **Widevine proxy via Google** | Direct, but requires Google partnership |
| **Bitmovin DRM** | Bundled with Bitmovin encoding; simplest if already using Bitmovin |
| **AWS Speke** | API standard for key exchange, works with MediaConvert |

Using a multi-DRM provider dramatically reduces implementation complexity — they handle
key management, Widevine/FairPlay/PlayReady interop, and token validation.

---

## Player DRM Configuration

### Shaka Player (Widevine + PlayReady + FairPlay)

```javascript
const player = new shaka.Player();
await player.attach(video);

player.configure({
  drm: {
    servers: {
      'com.widevine.alpha': 'https://license.axinom.com/license/widevine',
      'com.microsoft.playready': 'https://license.axinom.com/license/playready',
    },
    advanced: {
      'com.widevine.alpha': {
        // Robustness — set to '' for testing, 'SW_SECURE_CRYPTO' for L3, 'HW_SECURE_ALL' for L1
        videoRobustness: 'SW_SECURE_CRYPTO',
        audioRobustness: 'SW_SECURE_CRYPTO',
      },
    },
  },
});

// Add license token via request filter
player.getNetworkingEngine().registerRequestFilter((type, request) => {
  if (type === shaka.net.NetworkingEngine.RequestType.LICENSE) {
    request.headers['X-AxDRM-Message'] = YOUR_LICENSE_TOKEN;
  }
});

await player.load('https://cdn.example.com/manifest.mpd');
```

**FairPlay with Shaka:** Requires additional configuration due to Apple's proprietary
license exchange format:

```javascript
player.configure({
  drm: {
    servers: {
      'com.apple.fps.1_0': 'https://license.axinom.com/license/fairplay',
    },
  },
});

// FairPlay requires a certificate
const response = await fetch('https://your-fps-server.com/certificate');
const certificate = await response.arrayBuffer();
player.configure('drm.advanced.com\\.apple\\.fps\\.1_0.serverCertificate',
  new Uint8Array(certificate));
```

### hls.js (Widevine / PlayReady via EME)

```javascript
const hls = new Hls({
  emeEnabled: true,
});

hls.on(Hls.Events.KEY_LOADING, (event, data) => {
  // Modify license request if needed
});

// Configure EME
hls.config.drmSystems = {
  'com.widevine.alpha': {
    licenseUrl: 'https://license.axinom.com/license/widevine',
    // Optional: custom headers
    licenseXhrSetup: (xhr, url) => {
      xhr.setRequestHeader('X-AxDRM-Message', YOUR_TOKEN);
    },
  },
};

hls.loadSource(src);
hls.attachMedia(video);
```

**Note:** hls.js FairPlay support is limited. For Safari + FairPlay, fall back to native
`AVPlayer` or Bitmovin Player.

### AVPlayer + FairPlay (iOS / tvOS)

Apple's FairPlay Streaming (FPS) requires implementing `AVAssetResourceLoaderDelegate`.

```swift
class FairPlayResourceLoader: NSObject, AVAssetResourceLoaderDelegate {

    let licenseServerURL = "https://license.axinom.com/license/fairplay"
    let certificateURL = "https://your-fps-server.com/certificate"

    func resourceLoader(
        _ resourceLoader: AVAssetResourceLoader,
        shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest
    ) -> Bool {

        guard let url = loadingRequest.request.url,
              url.scheme == "skd" else { return false }

        Task {
            do {
                // 1. Fetch FPS certificate
                let certResponse = try await URLSession.shared.data(from: URL(string: certificateURL)!)
                let certificate = certResponse.0

                // 2. Generate SPC (Server Playback Context)
                guard let contentKeyRequest = loadingRequest.streamingContentKeyRequestData(
                    forApp: certificate,
                    contentIdentifier: url.host!.data(using: .utf8)!,
                    options: nil
                ) else { return }

                // 3. Send SPC to license server, receive CKC
                var licenseRequest = URLRequest(url: URL(string: licenseServerURL)!)
                licenseRequest.httpMethod = "POST"
                licenseRequest.httpBody = contentKeyRequest
                licenseRequest.setValue(YOUR_TOKEN, forHTTPHeaderField: "X-AxDRM-Message")

                let (ckc, _) = try await URLSession.shared.data(for: licenseRequest)

                // 4. Return CKC to AVPlayer
                loadingRequest.dataRequest?.respond(with: ckc)
                loadingRequest.finishLoading()
            } catch {
                loadingRequest.finishLoading(with: error)
            }
        }

        return true
    }
}
```

### ExoPlayer + Widevine (Android)

```kotlin
import androidx.media3.common.MediaItem
import androidx.media3.common.util.Util

val drmConfig = MediaItem.DrmConfiguration.Builder(C.WIDEVINE_UUID)
    .setLicenseUri("https://license.axinom.com/license/widevine")
    .setLicenseRequestHeaders(
        mapOf("X-AxDRM-Message" to YOUR_TOKEN)
    )
    .build()

val mediaItem = MediaItem.Builder()
    .setUri("https://cdn.example.com/manifest.mpd")
    .setDrmConfiguration(drmConfig)
    .build()

player.setMediaItem(mediaItem)
player.prepare()
```

---

## Token-Based License Authentication

The license server must validate that the player requesting a key is authorised.
Standard pattern: your backend issues a short-lived JWT or DRM-specific token.

```javascript
// Backend: issue a license token (example with Axinom)
// The token encodes: content ID, expiry, allowed actions, viewer ID
// Each multi-DRM provider has their own token format — use their SDK

// Frontend: attach the token to license requests
// (See Shaka / hls.js / ExoPlayer examples above)
```

**Never put license tokens in your frontend source code or HTML.** Issue them server-side,
tied to the authenticated user session.

---

## DRM + Commercial Encoding Vendors

All commercial encoding vendors handle DRM packaging. Key differences:

| Vendor | DRM Packaging | License Server | Notes |
|--------|--------------|----------------|-------|
| **Bitmovin** | Built-in | Partner (Axinom, EZDRM, BuyDRM) or SPEKE | Simplest path if using Bitmovin encoder |
| **AWS MediaConvert** | SPEKE API | AWS ACM or third-party | Native AWS integration |
| **Mux** | Built-in | Managed by Mux | Widevine + FairPlay; PlayReady in beta |
| **Cloudflare Stream** | ❌ | None | Use signed URLs for basic protection instead |

---

## Common Mistakes

**Mixing encryption schemes:** CENC (cbcs) is required for HLS on Apple devices; CENC (cenc/ctr)
for Widevine/PlayReady. Shaka Packager handles this automatically; verify your packager does too.

**Forgetting CORS on the license server:** The browser makes a cross-origin XHR to your license
server. Set `Access-Control-Allow-Origin` correctly.

**Long-lived tokens:** DRM tokens should expire (hours, not days). If a token is leaked, short
expiry limits the damage window.

**L1 vs L3 Widevine:** L1 (hardware-backed, required for HD+ on premium content) requires device
certification. Most desktop browsers are L3. Android devices with Widevine L1 support it natively.
