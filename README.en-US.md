

# Home KTV

**中文** | [English](README_EN.md)

## Interface Preview

| Mobile Songbook Homepage | Android TV Standby Screen | Library & Service Dashboard |
| --- | --- | --- |
| ![Mobile Songbook Homepage](docs/images/mobile-songbook.png) | ![Android TV Standby Screen](docs/images/tv-player.png) | ![Admin Dashboard](docs/images/admin-dashboard.png) |
| Search, categorize, favorite, queue songs, and control playback entirely from a mobile browser. | The actual standby screen displays the song-queue QR code, service status, and recommended songs. | View the library, transcoding tasks, and playback service status in one place. |

Home KTV is a LAN-based song-queue system running on a home NAS or Linux host. The TV handles playback, while family members use WeChat or a mobile browser to scan a QR code and access the song-queue page. The server manages the song library, queue, lyrics, playback history, and system settings.

The system consists of three clients:

- **Server**: Spring Boot, PostgreSQL, FFmpeg/FFprobe, WebSocket
- **Mobile**: Vue 3 H5 song-queue page and admin dashboard, no app installation required
- **TV**: Android TV client, based on Media3/ExoPlayer

> This project is designed for trusted home LANs. It does not provide public internet login or security measures, so do not expose it directly to the internet.

## Typical Usage Flow

1. Start Home KTV on your NAS or Linux host and scan/organize the local song library via the admin dashboard.
2. The Android TV client automatically discovers the LAN service. Once connected, it displays a song-queue QR code on the large screen.
3. Family members scan the QR code with WeChat or a mobile browser to join. They can search, favorite, and queue songs without installing any app.
4. The song queue, playback progress, lyrics, volume, and original/accompaniment track status sync in real-time between the TV and mobile devices.
5. After singing, users can view their recent performance history on their phones. Administrators can manage songs, artists, playlists, and transcoding tasks in the background.

## Version Updates

- **Source File Pipeline**: Re-evaluates files during scanning to determine if transcoding is needed. Compatible files are directly moved to the KTV library, and their original management records are cleared. Incompatible files remain in the original music management waiting for transcoding. Batch transcoding supports progress tracking and queue insertion. Once automatic cleanup finishes, you'll be prompted to rescan the source path.
- **KTV Library Management**: Song lists use paginated queries and a fixed action area. Metadata scraping supports full batches, pause/resume, detailed progress, automatic confidence score saving, manual review, manual editing, single-song re-matching, and cover image restoration.
- **Artist Database**: Artists are aggregated by standardized names, supporting gender status, batch AI analysis, and manual review. During name conflict review, representative songs are displayed for judgment.
- **Mobile Song-Queue Page**: Song lists uniformly display cover images. Language and category filters are promoted to primary entries. Artists can be filtered by gender, and songs can be added to existing playlists.
- **Themed Playlists**: Generates editable previews using the organized library metadata. Each playlist supports up to 100 songs; playlists with fewer than 100 can still be saved. AI-generated playlists are deletable.
- **Settings Hub**: Redesigned with a category sidebar, search, and right-side content area. Supports categories for basic config, AI models, import & transcoding, TV display, music metadata, and data maintenance, along with deep linking.
- **AI Configuration & Fallback**: Supports arbitrary OpenAI-compatible URLs and model IDs, model list detection, single/dual models, capability testing, and concurrency limits. If no key is configured or calls fail, parsing tasks with local rule alternatives automatically fall back; AI operations without equivalent local capabilities will prompt the admin to configure a model first.
- **Release & Upgrade**: The release pipeline only builds signed Release APKs, embedding both 32-bit and 64-bit installers into the Docker image. The admin dashboard displays release notes by version. Upon TV connection, users can download the appropriate installer for their device architecture and open the system installation screen.
- **Upgrade Safety**: Flyway V15 retains historical source records and avoids table clearing or bulk deletions. Migration safety tests block direct submissions of `DELETE FROM`, `TRUNCATE TABLE`, and `DROP TABLE/COLUMN`.

## Main Features

### Mobile Song-Queue & Remote Control

- Scan QR code with WeChat or browser, no registration needed
- Supports search by song title, artist, Chinese characters, full pinyin, and pinyin initials
- Queue songs, prioritize songs, delete personal songs, smart shuffle, and multi-user queue
- Play, pause, repeat, skip, adjust volume, and switch between original/accompaniment tracks
- Favorites, recent performances, one-click replay, trending charts, and public playlists
- Line-by-line LRC and enhanced word-by-word LRC lyrics
- Live sound effects like applause, cheering, booing, and clinking glasses

### Android TV Playback

- Plays common media formats like MP4, MKV, MPEG, MP3, FLAC
- Supports MPEG-2, MP2, and dual-track KTV videos
- Switches between original/accompaniment tracks without reloading
- Dual-line lyrics showing current and next lines, with continuous color scanning highlight on the current line
- Remote control for playback, queue, volume, and original/accompaniment track
- Reconnection on disconnect, state recovery, standby slideshow, and anti-burn-in micro-panning
- Playback page displays a "WeChat Scan to Queue" QR code
- Supports Android 8.0 (API 26) and above

### Library Management

- Raw material analysis, MD5 deduplication, direct copy, and batch transcoding
- Uses FFprobe to identify containers, codecs, duration, resolution, and audio tracks
- Automatically reads media tags, cover images, and same-name sidecar LRC files
- Identifies KTV videos, standard MVs, and pure audio
- Audio track tag correction, song editing, re-parsing, and handling of invalid files
- AI-assisted classification, themed playlists, and queue statistics
- PostgreSQL persistence with database backup/restore

## System Architecture

```text
Mobile Browser / WeChat
        │ HTTP + WebSocket
        ▼
Home KTV Server ───── PostgreSQL
        │
        ├── /source-music  Raw source directory
        ├── /music         Playable library directory
        │
        └── Android TV     Video, audio tracks, lyrics, and control
```

The server exposes the following default endpoints:

| Purpose | Address |
| --- | --- |
| Mobile Song-Queue | `http://<host_ip>:8080/m` |
| Admin Dashboard | `http://<host_ip>:8080/m/admin` |
| Health Check | `http://<host_ip>:8080/api/health` |

## Quick Start

### Requirements

- NAS, Linux host, or Docker Desktop supporting Docker Compose
- At least 1 GB of available RAM recommended
- Mobile device, Android TV, and server must be on the same LAN
- Android TV 8.0 (API 26) or higher

### 1. Configure Directories and Passwords

```bash
git clone <repository_url>
cd home-ktv
cp .env.example .env
```

Edit `.env` and verify at least the following configurations:

```dotenv
KTV_SOURCE_MUSIC_DIR=/volume1/home-ktv/source-music
KTV_MUSIC_DIR=/volume1/home-ktv/music
KTV_DB_PASSWORD=ReplaceWithStrongPassword
```

- `KTV_SOURCE_MUSIC_DIR`: Directory for raw, unprocessed video and audio.
- `KTV_MUSIC_DIR`: Directory for files that are directly copied or fully transcoded and ready for playback.

Do not configure both directories to the same path. The server writes processing results to the library directory, so ensure the container has write permissions.

### 2. Start the Service

It is recommended to directly pull the multi-architecture image published via GitHub Actions, avoiding compilation on your NAS or host:

```bash
docker compose -f docker-compose.prebuilt.yml up -d --pull always --wait
```

Uses `ghcr.io/zhayinggang/ktv-home:latest` by default. For production, set `KTV_RELEASE_IMAGE` to a specific release tag in `.env` to prevent automatic `latest` changes.

Use the following to build from source:

```bash
docker compose up -d --build --wait
```

Verify container health:

```bash
docker compose ps
curl http://127.0.0.1:${KTV_HTTP_PORT:-8080}/api/health
```

Default ports:

- TCP `8080`: H5, admin dashboard, API, WebSocket, and media streams
- UDP `18888`: Android TV LAN auto-discovery

Ensure your NAS firewall allows these ports; if using custom ports, refer to `.env`.

### 3. Import Songs

1. Place raw songs into `KTV_SOURCE_MUSIC_DIR`.
2. Open `http://<host_ip>:8080/m/admin`.
3. Execute "Scan Source Path" on the dashboard.
4. Compatible files are automatically direct-copied to the library; incompatible files go to the pending transcoding list.
5. Perform single, selected, or batch transcoding in "Original Music Management".
6. Verify song titles, artists, media types, and original/accompaniment tracks in "KTV Library".

Scanning only handles analysis, deduplication, and direct copying; it does not automatically trigger time-consuming transcoding. Batch transcoding progress can be viewed in the admin dashboard, and you can insert specific songs to be processed next while a task is running.

Auto-listening for source directories is disabled by default. Enable it via the "Source Directory Auto-Scan" toggle in the admin dashboard "System Settings". When disabled, scanning only occurs when manually clicking "Scan Source Path"; it is not configured via Compose or environment variables.

After verifying the import results, click "Auto Cleanup" next to the batch transcoding button to free up space in the source directory. The system only deletes source files that have been successfully imported, have valid library records, and have corresponding output files in the library. Files pending transcoding, failed, duplicates, unrecognized, missing library outputs, or failing path validation are retained. Auto cleanup cannot run during transcoding. Once finished, return to the dashboard and re-run "Scan Source Path" to sync the latest files from the source directory.

### 4. Install Android TV Client

The official release image embeds version-matched 32-bit (`armeabi-v7a`) and 64-bit (`arm64-v8a`) Release APKs. The release pipeline generates `versionName` from the release tag and a monotonically increasing `versionCode` from the GitHub Actions run number, writing the same version info to the server image and both APKs.

The backend returns version info, release notes, and installer links for both architectures via `GET /api/release`. The release note ID equals the version number by default, so new versions will always show upon release. "Remind Me Later" in the admin dashboard only hides the notice within the current browser session, while "Mark as Read" saves the notice ID in the current browser until the ID or version number changes. Notices only pop up if enabled and at least one APK exists in the image; development images without embedded APKs will not show invalid download notices.

Default notices not only prompt to download the TV APK but also remind admins: after upgrading, go to "Original Music Management" and execute "Auto Cleanup", then return to the dashboard to rescan the original music path. Notice configuration is released with `application.yml` inside the image, independent of user updates to `docker-compose.yml` or `.env`. Pulling the new image automatically grants the new version number and notice content.

The TV client checks `versionCode` every time it successfully connects to the server. If versions mismatch, it selects the installer based on the device ABI. After the user clicks "Go to Download", the client verifies the download size, requests "Install Unknown Apps" permission, and opens the system installer. If query or download fails, it won't affect playback, and you can retry the download. Installers are also directly accessible via:

```text
http://<host_ip>:8080/api/release/tv/apk/armeabi-v7a
http://<host_ip>:8080/api/release/tv/apk/arm64-v8a
```

Downloaded filenames will be `home-ktv-tv-<version>-armeabi-v7a.apk` and `home-ktv-tv-<version>-arm64-v8a.apk`.

After the initial Release APK installation, subsequent versions must use the same signing certificate. Historical Debug APKs use a `.debug` package name and a different signature, so they can't be overwritten directly by the Release APK. Uninstall the Debug version first before installing the Release version. Uninstalling clears the saved server address on the TV, but song and server data remain unaffected.

Local Debug APK build requires JDK 17 and Android SDK:

```bash
cd android-tv
./gradlew testDebugUnitTest assembleDebug
```

APK output location:

```text
android-tv/app/build/outputs/apk/debug/app-debug.apk
```

Install via ADB:

```bash
adb connect <tv_ip>:5555
adb install -r android-tv/app/build/outputs/apk/debug/app-debug.apk
```

You can also install via USB drive or TV file manager. On first launch, it attempts auto-discovery; if that fails, manually enter `<host_ip>:8080`. Some TV boxes require additional permissions for "Install Unknown Apps", auto-start, and background running.

### 5. Start Queuing Songs

Once the TV connects successfully, a QR code will appear. Scan it with WeChat to access the song-queue page on your phone. Select songs, and the TV will auto-play them. The queue, playback status, lyrics, and remote control operations sync in real-time via WebSocket.

## Media & Lyrics

### File Naming

The system prioritizes media tags. If tags are missing, it infers the artist and song title from the filename. Recommended format:

```text
Artist - Song Title.mp4
Artist - Song Title.mkv
Artist - Song Title.mp3
```

Example:

```text
source-music/
├── Jay Chou - Blue and White Porcelain.mp4
├── S.H.E - Super Star.mpg
└── Beyond - Boundless Oceans, Vast Skies.mkv
```

When multiple versions of the same song exist, dual-track KTV videos take precedence over standard MVs and pure audio.

### Dual-Track Convention

Recommended video audio track order:

1. Original Vocals
2. Accompaniment

It's also recommended to write track titles as `Original Vocals` and `Accompaniment` with the language tag `zho`. The system auto-detects the accompaniment track; if misidentified, you can correct and save it on the mobile remote page.

The repository provides a basic channel-subtraction script to generate dual-track files for stereo MVs meeting channel conditions:

```bash
./scripts/make_ktv_mv.sh input.mp4 output.mp4
```

This script does not use AI vocal separation. Its effectiveness depends on the original audio's channel mixing and cannot replace official accompaniments or professional stem extraction.

### Lyrics Sidecar Files

Lyrics files share the same base name as the media and reside in the same directory:

```text
Jay Chou - Blue and White Porcelain.mp4
Jay Chou - Blue and White Porcelain.lrc
```

Supports standard line-by-line LRC:

```text
[00:12.50]A little yellow flower from the story
```

Supports enhanced LRC with word-level timestamps:

```text
[00:12.50]<00:12.50>A<00:12.80>l<00:13.10>i<00:13.35>t<00:13.60>l<00:13.90>e
```

Enhanced LRC displays as a continuously color-scanning sentence on the TV, with word-level sync on the mobile lyrics page.

## Configuration Reference

Common Environment Variables:

| Variable | Default | Description |
| --- | --- | --- |
| `KTV_SOURCE_MUSIC_DIR` | `./source-music` | Host path for raw source materials |
| `KTV_MUSIC_DIR` | `./music` | Host path for playable library directory |
| `KTV_DATA_DIR` | `./data` | Host path for application data |
| `KTV_PG_DIR` | `./postgres` | Host path for PostgreSQL data |
| `KTV_HTTP_PORT` | `8080` | Web, API, WebSocket, and media stream port |
| `KTV_DISCOVERY_UDP_PORT` | `18888` | TV auto-discovery UDP port |
| `KTV_DISCOVERY_NAME` | `Home KTV` | Name shown in TV discovery list |
| `KTV_DB_NAME` | `ktv` | PostgreSQL database name |
| `KTV_DB_USER` | `ktv` | PostgreSQL username |
| `KTV_DB_PASSWORD` | `ktv` | PostgreSQL password, must be changed for production |
| `KTV_IMAGE_REGISTRY` | `docker.m.daocloud.io` | Docker base image registry prefix |
| `KTV_APP_IMAGE` | `home-ktv:latest` | Application image name |
| `KTV_RELEASE_IMAGE` | `ghcr.io/zhayinggang/ktv-home:latest` | GitHub container image used for prebuilt Compose |
| `JAVA_TOOL_OPTIONS` | `-XX:MaxRAMPercentage=70 -Xmx512m` | Container JVM memory parameters |

The QR code defaults to the LAN host address used by the TV to access the server. If a reverse proxy or multiple network interfaces exist, you can set the "Display Address" in the admin dashboard, e.g., `192.168.1.10:8080`.

## Hardware Transcoding

CPU transcoding is used by default. Linux hosts can pass through VAAPI devices as follows:

> **Validation Scope:** VAAPI H.264 / HEVC hardware encoding has currently only been validated on Intel integrated graphics, using the Intel `iHD` driver. AMD VAAPI and Rockchip RK MPP have not yet been validated on actual hardware, related Compose configs only indicate device passthrough support and do not guarantee that prebuilt images can directly enable hardware encoding.

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.hardware.yml \
  up -d --build --wait
```

The host must provide `/dev/dri`. Intel devices also require `iHD_drv_video.so` inside the container; official prebuilt AMD64 images install `intel-media-driver`. After startup, check and enable hardware acceleration in the admin dashboard "System Settings"; the system will reject saving if the device, driver, permissions, or encoder are unavailable.

For Rockchip devices, use:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.rockchip.yml \
  up -d --build --wait
```

The host must provide `/dev/mpp_service`, and FFmpeg must include the corresponding RK MPP encoder. Generic images do not guarantee inclusion of `h264_rkmpp` or `hevc_rkmpp`.

## AI Configuration & Fallback

AI is disabled by default and does not affect scanning, transcoding, importing, queuing, or playback. It is recommended to configure it via "System Settings → AI Models" in the admin dashboard; alternatively, use environment variables to connect to any OpenAI-compatible Chat Completions service:

```dotenv
KTV_AI_ENABLED=true
KTV_AI_BASE_URL=https://ai.example/v1
KTV_AI_API_KEY=your_api_key
KTV_AI_BULK_MODEL=your-model-id
KTV_AI_REASONING_MODEL=
KTV_AI_JSON_MODE=AUTO
KTV_AI_BULK_CONCURRENCY=2
KTV_AI_REASONING_CONCURRENCY=1
KTV_AI_AUTO_APPLY_CONFIDENCE=0.90
```

Model IDs are not restricted to a fixed enumeration. If the enhancement model is left empty, it reuses the batch model. The backend attempts to fetch the model list and detect auth, Chat Completions, and JSON output capabilities. Services without list or JSON Mode support can still be manually configured and will auto-fallback.

API keys saved in the admin dashboard are encrypted with AES-256-GCM. The API only returns configuration status and the last few characters. The master key prioritizes `KTV_CONFIG_MASTER_KEY`; otherwise, it generates `secrets/config.key` in the data directory. Do not delete or lose this file, or saved API keys will become undecryptable.

If AI is not configured or calls timeout, hit rate limits, or fail, local tags, filenames, lyrics tags, and directory parsing continue to function without blocking imports. Operations lacking equivalent local judgment capabilities, such as natural language themed playlists or artist gender inference, will prompt the admin to configure a model first and will not fabricate results.

Redeploy after modifying `.env`:

```bash
docker compose up -d --build
```

Keys should only be saved in the local `.env` or controlled secrets, not committed to the repository.

## Daily Operations

### Updates

```bash
git pull
docker compose up -d --build --wait
```

### Stop & Start

```bash
docker compose stop
docker compose start
```

Remove containers but keep data volumes:

```bash
docker compose down
```

Do not run `docker compose down -v` when data preservation is required.

### Logs

```bash
docker compose logs -f ktv
docker compose logs -f db
```

### Backup & Restore

```bash
./scripts/backup.sh /volume1/backup/home-ktv
```

Restore a specific backup:

```bash
./scripts/restore.sh \
  /volume1/backup/home-ktv/home-ktv-YYYYMMDD-HHMMSS.dump \
  --yes
```

Database backups include song metadata, settings, playlists, queues, and playback history, but not raw media files. The `source-music` and `music` directories require your NAS's native backup solution. Flyway upgrades execute automatically on app startup; it is still recommended to back up the database and media directories before official updates. Current migrations retain historical source records and do not batch-delete business data via upgrade scripts.

## Local Development

The development environment requires Node.js 20+, JDK 21, JDK 17, Docker, and Android SDK.

```bash
# PostgreSQL
docker compose -f docker-compose.dev.yml up -d

# Backend, JDK 21
cd backend
./mvnw spring-boot:run

# H5
cd h5
npm install
npm run dev

# Android TV, JDK 17
cd android-tv
./gradlew testDebugUnitTest assembleDebug
```

Test commands:

```bash
cd backend && ./mvnw test
cd h5 && npm test
cd android-tv && ./gradlew testDebugUnitTest
```

Directory Structure:

```text
backend/      Spring Boot server, database, scanning, transcoding, and real-time control
h5/           Vue 3 mobile song-queue client and admin dashboard
android-tv/   Kotlin Android TV client
scripts/      Backup, restore, and media helper scripts
```

## FAQ

### Mobile QR Code Fails to Open

- Ensure the mobile device and server are on the same LAN.
- Ensure the QR code address is not a Docker `172.x` container IP.
- Set the correct "Display Address" in the admin dashboard.
- Check NAS firewall and TCP ports.

### TV Cannot Auto-Discover Server

- Ensure UDP `18888` is allowed.
- Ensure AP isolation or guest network isolation is disabled on the router.
- Manually enter `<host_ip>:8080` on the TV's initial setup page.

### Original Vocals and Accompaniment Swapped

Click "Are original vocals and accompaniment swapped?" on the mobile remote page. The system will correct the accompaniment track tag for the current file and save it.

### Video Won't Play or Stutters

- Check format analysis results in Original Music Management.
- Transcode incompatible files.
- Check NAS CPU, disk, and network usage.
- Linux hosts can try enabling VAAPI or RK MPP hardware transcoding.

### Lyrics Not Displaying

- Ensure the LRC base name exactly matches the media file.
- Verify timestamp format is `[mm:ss.xx]`.
- Rescan after editing lyrics; the system supports sidecar lyrics updates with matching names.

## Known Limitations

- The system is only for trusted LANs and should not be exposed to the public internet.
- Accompaniments generated via AI vocal separation or channel subtraction may retain lead vocals and cannot match official mastering quality.
- Bluetooth microphone latency and audio quality depend on the TV box firmware; prioritize USB or wired devices for real-time singing.
- Audio track ordering varies across different KTV videos; manual spot-checking is recommended after initial import.
- Android TV auto-start and background process retention may require additional permissions from the device manufacturer.

## License & Media Responsibility

Project code is licensed under the [MIT License](LICENSE), permitting free use, modification, distribution, and commercial use, provided the license and copyright notice are retained.

Only import and play media you have the right to use. Home KTV does not provide, download, or distribute song, MV, or accompaniment resources.

When reporting issues, please include your NAS OS, Android TV model, Android version, media format, and relevant logs. Remove private information like IPs, passwords, and keys.
