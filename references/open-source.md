# Open-Source Video Stack Reference

Tools covered: **ffmpeg**, **Shaka Player**, **hls.js**, **AVPlayer (iOS)**, **ExoPlayer (Android)**

---

## VOD Encoding with ffmpeg

### Standard ABR Ladder (HLS + DASH)

The most common starting point: one source file → multiple renditions → HLS/DASH manifests.

```bash
# Inputs
INPUT="source.mp4"
OUTPUT_DIR="output"
mkdir -p "$OUTPUT_DIR"

# Generate H.264 ABR renditions
ffmpeg -i "$INPUT" \
  -filter_complex \
    "[0:v]split=4[v1][v2][v3][v4];\
     [v1]scale=1920:1080[v1out];\
     [v2]scale=1280:720[v2out];\
     [v3]scale=854:480[v3out];\
     [v4]scale=640:360[v4out]" \
  \
  -map "[v1out]" -c:v:0 libx264 -b:v:0 4500k -maxrate:v:0 4950k -bufsize:v:0 9000k -preset slow -g 48 -sc_threshold 0 \
  -map "[v2out]" -c:v:1 libx264 -b:v:1 2500k -maxrate:v:1 2750k -bufsize:v:1 5000k -preset slow -g 48 -sc_threshold 0 \
  -map "[v3out]" -c:v:2 libx264 -b:v:2 1200k -maxrate:v:2 1320k -bufsize:v:2 2400k -preset slow -g 48 -sc_threshold 0 \
  -map "[v4out]" -c:v:3 libx264 -b:v:3 600k  -maxrate:v:3 660k  -bufsize:v:3 1200k -preset slow -g 48 -sc_threshold 0 \
  \
  -map 0:a -c:a:0 aac -b:a:0 192k \
  -map 0:a -c:a:1 aac -b:a:1 128k \
  -map 0:a -c:a:2 aac -b:a:2 96k  \
  -map 0:a -c:a:3 aac -b:a:3 64k  \
  \
  -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2 v:3,a:3" \
  -master_pl_name master.m3u8 \
  -f hls \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_flags independent_segments \
  -hls_segment_type mpegts \
  -hls_segment_filename "$OUTPUT_DIR/stream_%v/seg_%03d.ts" \
  "$OUTPUT_DIR/stream_%v/playlist.m3u8"
```

**Key flag explanations:**
- `-g 48 -sc_threshold 0`: Fixed GOP of 48 frames (2s at 24fps). `sc_threshold 0` disables scene-cut detection so segment boundaries are predictable — important for ABR switching.
- `-preset slow`: Better compression at cost of CPU time. Use `medium` for faster jobs, `veryslow` for archival.
- `-maxrate`/`-bufsize`: CBR-like constraint. `bufsize` = 2× `maxrate` is a standard VBV buffer.
- `independent_segments`: Each segment is independently decodable — required for proper seeking.

### CMAF / fMP4 segments (preferred for DASH + HLS dual output)

```bash
ffmpeg -i "$INPUT" \
  -filter_complex "[0:v]split=3[v1][v2][v3];[v1]scale=1280:720[hd];[v2]scale=854:480[sd];[v3]scale=640:360[lo]" \
  \
  -map "[hd]" -c:v:0 libx264 -b:v:0 2500k -g 48 -keyint_min 48 -sc_threshold 0 \
  -map "[sd]" -c:v:1 libx264 -b:v:1 1200k -g 48 -keyint_min 48 -sc_threshold 0 \
  -map "[lo]" -c:v:2 libx264 -b:v:2 600k  -g 48 -keyint_min 48 -sc_threshold 0 \
  -map 0:a   -c:a aac -b:a 128k \
  \
  -f dash \
  -seg_duration 6 \
  -use_timeline 1 \
  -use_template 1 \
  -init_seg_name "init_\$RepresentationID\$.mp4" \
  -media_seg_name "chunk_\$RepresentationID\$_\$Number%05d\$.m4s" \
  -hls_playlist 1 \
  "$OUTPUT_DIR/manifest.mpd"
```

Adding `-hls_playlist 1` outputs both `manifest.mpd` and HLS playlists from a single pass.

### H.265/HEVC (for bandwidth savings ~40% vs H.264)

```bash
# Replace libx264 with libx265, adjust bitrates down ~40%
-c:v libx265 -b:v:0 2700k -preset slow -x265-params "keyint=48:min-keyint=48:scenecut=0"
```

**Caveat:** H.265 requires checking browser/device support. Use with CMAF/fMP4 segments for best compatibility. Not supported in older browsers without MSE polyfills.

### Audio-only track for low-bandwidth fallback

```bash
ffmpeg -i "$INPUT" -vn -c:a aac -b:a 64k \
  -f hls -hls_time 6 -hls_playlist_type vod \
  "$OUTPUT_DIR/audio_only/playlist.m3u8"
```

Add this to your master playlist under `#EXT-X-STREAM-INF:BANDWIDTH=64000,CODECS="mp4a.40.2"`.

### Thumbnail sprite generation

```bash
ffmpeg -i "$INPUT" \
  -vf "fps=1/10,scale=160:90,tile=10x10" \
  -frames:v 1 \
  "$OUTPUT_DIR/thumbnails/sprite_%03d.jpg"
```

---

## Packaging & Delivery

### Static hosting (S3 / GCS / Azure Blob)

For VOD, serve segments directly from object storage with a CDN in front.

**S3 CORS config** (required for cross-origin video playback):
```json
[{
  "AllowedHeaders": ["*"],
  "AllowedMethods": ["GET", "HEAD"],
  "AllowedOrigins": ["https://yourdomain.com"],
  "ExposeHeaders": ["Content-Length", "Content-Range"]
}]
```

Set `Cache-Control: max-age=31536000, immutable` on segment files.
Set `Cache-Control: max-age=5` on manifest files (`.m3u8`, `.mpd`).

### MP4Hls / Bento4 for DASH packaging

For more control over DASH manifests (e.g., adding subtitles, multiple audio tracks):

```bash
# Install Bento4
# https://www.bento4.com/

mp4fragment --fragment-duration 6000 input.mp4 fragmented.mp4
mp4dash --use-segment-timeline fragmented.mp4 -o output/
```

---

## Players

### hls.js (web, HLS)

Best for: HLS playback in browsers that don't natively support it (i.e., everything except Safari).

```html
<script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
<video id="video" controls></video>

<script>
  const video = document.getElementById('video');
  const src = 'https://cdn.example.com/output/master.m3u8';

  if (Hls.isSupported()) {
    const hls = new Hls({
      // Tune ABR behaviour
      startLevel: -1,          // -1 = auto
      capLevelToPlayerSize: true,
      maxBufferLength: 30,
      maxMaxBufferLength: 600,
    });
    hls.loadSource(src);
    hls.attachMedia(video);
  } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
    // Safari native HLS
    video.src = src;
  }
</script>
```

**DRM with hls.js:** hls.js supports Widevine/PlayReady via EME. See `drm.md` § hls.js DRM.

### Shaka Player (web, HLS + DASH)

Best for: DASH playback, multi-DRM (Widevine + PlayReady + FairPlay), subtitle-heavy content.

```html
<script src="https://ajax.googleapis.com/ajax/libs/shaka-player/4.7.11/shaka-player.compiled.js"></script>
<video id="video" autoplay controls></video>

<script>
  shaka.polyfill.installAll();

  async function init() {
    if (!shaka.Player.isBrowserSupported()) {
      console.error('Browser not supported');
      return;
    }

    const video = document.getElementById('video');
    const player = new shaka.Player();
    await player.attach(video);

    // Error handling
    player.addEventListener('error', (event) => {
      console.error('Shaka error', event.detail);
    });

    // Load manifest (HLS or DASH)
    await player.load('https://cdn.example.com/output/manifest.mpd');
  }

  document.addEventListener('DOMContentLoaded', init);
</script>
```

For DRM config, add a `drm` key to `player.configure({})`. See `drm.md` § Shaka DRM.

### AVPlayer (iOS / tvOS, Swift)

Native HLS playback. Apple's preferred player — handles HLS natively including FairPlay DRM.

```swift
import AVFoundation
import AVKit

class VideoViewController: UIViewController {

    var player: AVPlayer?
    var playerViewController: AVPlayerViewController?

    override func viewDidLoad() {
        super.viewDidLoad()
        setupPlayer()
    }

    func setupPlayer() {
        guard let url = URL(string: "https://cdn.example.com/output/master.m3u8") else { return }

        let asset = AVURLAsset(url: url)
        let playerItem = AVPlayerItem(asset: asset)

        // Optional: configure preferred peak bit rate
        playerItem.preferredPeakBitRate = 2_500_000 // 2.5 Mbps max

        player = AVPlayer(playerItem: playerItem)

        let playerVC = AVPlayerViewController()
        playerVC.player = player
        playerViewController = playerVC

        addChild(playerVC)
        view.addSubview(playerVC.view)
        playerVC.view.frame = view.bounds
        playerVC.didMove(toParent: self)

        player?.play()
    }
}
```

For FairPlay DRM, implement `AVAssetResourceLoaderDelegate`. See `drm.md` § FairPlay.

### ExoPlayer / Media3 (Android)

Google's recommended player for Android. Now part of **AndroidX Media3**.

```kotlin
// build.gradle.kts
implementation("androidx.media3:media3-exoplayer:1.3.1")
implementation("androidx.media3:media3-exoplayer-hls:1.3.1")
implementation("androidx.media3:media3-exoplayer-dash:1.3.1")
implementation("androidx.media3:media3-ui:1.3.1")
```

```kotlin
import androidx.media3.common.MediaItem
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.ui.PlayerView

class VideoActivity : AppCompatActivity() {

    private var player: ExoPlayer? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_video)

        val playerView = findViewById<PlayerView>(R.id.player_view)
        val player = ExoPlayer.Builder(this).build()

        playerView.player = player

        val mediaItem = MediaItem.fromUri("https://cdn.example.com/output/manifest.mpd")
        // Or for HLS:
        // val mediaItem = MediaItem.fromUri("https://cdn.example.com/output/master.m3u8")

        player.setMediaItem(mediaItem)
        player.prepare()
        player.play()

        this.player = player
    }

    override fun onStop() {
        super.onStop()
        player?.release()
        player = null
    }
}
```

For Widevine DRM, add `MediaItem.DrmConfiguration`. See `drm.md` § Widevine/ExoPlayer.

---

## Subtitles & Closed Captions

### WebVTT generation from SRT

```bash
ffmpeg -i subtitles.srt subtitles.vtt
```

### Embed WebVTT in HLS manifest

Add to each rendition playlist:
```
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="English",DEFAULT=YES,AUTOSELECT=YES,FORCED=NO,LANGUAGE="en",URI="subtitles_en.m3u8"
```

Reference the group in `#EXT-X-STREAM-INF`:
```
#EXT-X-STREAM-INF:BANDWIDTH=2500000,SUBTITLES="subs"
stream_0/playlist.m3u8
```

---

## Operational Considerations

**When open-source starts to strain:**
- Encoding queue management: consider adding a job queue (Bull/BullMQ, Celery) around ffmpeg workers
- Multi-instance scaling: ffmpeg is single-job-per-process; scale horizontally with containers
- Egress costs: object storage egress can surprise you at scale — model CDN + storage costs early
- DRM key management: running your own Widevine proxy requires a Google license (not free at scale)
- Smart TV/STB support: hls.js and Shaka do not run on all TV platforms; see `vendors.md` for where commercial players add value
