# RTSP/RTMP Streaming with MediaMTX

Streams live video into OBS Studio using Docker Compose and MediaMTX. Supports a synthetic test signal via FFmpeg, real sources such as a DJI Neo drone or iPhone, and 3D printer cameras.

## How it works

```
Publisher (FFmpeg / DJI Neo / iPhone) → MediaMTX (broker) → OBS Studio (consumer)
                                              :8554  RTSP
                                              :1935  RTMP
                                              :8888  HLS
                                              :8889  WebRTC  ← lowest latency
```

MediaMTX accepts streams over RTSP and RTMP, and re-serves them over HLS and WebRTC. OBS connects via WebRTC Browser Source for lowest latency.

## Prerequisites

- Docker Desktop (Mac) or Docker + Docker Compose (Ubuntu)
- OBS Studio

## File structure

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Base MediaMTX config — shared between local and production |
| `docker-compose.override.yml` | Local Mac overrides — FFmpeg test signal and static IP workaround — applied automatically |
| `docker-compose.prod.yml` | Production Ubuntu overrides — no FFmpeg, uses `mediamtx.prod.yml` |
| `mediamtx.yml` | Local MediaMTX config — no authentication |
| `mediamtx.prod.yml` | Production MediaMTX config — with authentication |

## Local (Mac)

```bash
docker compose up
```

The override file is applied automatically. FFmpeg publishes a color bar test signal to the `test` path.

## Production (Ubuntu)

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Before deploying, set strong passwords in `mediamtx.prod.yml` — replace `change-me` and `change-me-admin`.

## Connect OBS

Use **Browser Source** with the WebRTC endpoint for lowest latency (under 1 second):

1. Sources → **+** → **Browser Source**
2. **URL:** `http://localhost:8889/neo`
3. **Width:** 1920, **Height:** 1080

Replace `localhost` with the server's public IP when connecting to the production server.

## All endpoints

| URL | Protocol | Latency |
|-----|----------|---------|
| `rtsp://127.0.0.1:8554/{path}` | RTSP | low |
| `rtmp://127.0.0.1:1935/{path}` | RTMP | low |
| `http://localhost:8888/{path}` | HLS | 3–6 s |
| `http://localhost:8889/{path}` | WebRTC | < 1 s |
| `http://localhost:9997/v3/paths/list` | API | — |

## FFmpeg test signal (local only)

The test signal publishes a 720p SMPTE color bar to the `test` path:

- OBS Browser Source: `http://localhost:8889/test`
- VLC: `vlc rtsp://127.0.0.1:8554/test`

## VLC

```bash
/Applications/VLC.app/Contents/MacOS/VLC rtsp://127.0.0.1:8554/{path}
```

## Connecting a DJI Neo drone

The DJI Neo pushes RTMP via the DJI Fly app.

1. Find your Mac's WiFi IP:
   ```bash
   ipconfig getifaddr en1
   ```
2. In DJI Fly: tap **···** → **Live Stream** → **Custom RTMP**
3. Set the URL to `rtmp://192.168.x.x:1935/neo`
4. In OBS Browser Source: `http://localhost:8889/neo` (1920×1080)

If DJI Fly asks for a separate stream key, leave it empty or use any value — it will be appended to the path.

## Streaming from an iPhone

Use **Larix Broadcaster** (free on the App Store):

1. Install Larix Broadcaster
2. Go to **Settings → Connections → +**
3. Set URL to `rtmp://192.168.x.x:1935/iphone`
4. Tap **Save** and hit the record button

In OBS Browser Source: `http://localhost:8889/iphone` (1920×1080)

Your iPhone and Mac must be on the same WiFi network.

## Streaming from a Bambu Lab P1S 3D printer

The P1S has a built-in camera that streams over RTSPS (RTSP over TLS). MediaMTX pulls from the printer and re-serves it to OBS.

**What you need:**
- Printer's IP address (find it in Bambu Handy or your router)
- Access code (on the printer: Settings → Network → Access Code)

Add a path to `mediamtx.yml`:

```yaml
paths:
  p1s:
    source: rtsps://bblp:[access-code]@[printer-ip]:322/streaming/live/1
    sourceTLSInsecureSkipVerify: true
```

In OBS Browser Source: `http://localhost:8889/p1s` (1920×1080)

Works regardless of whether the printer is in LAN mode. The printer and Mac must be on the same network.

## Production authentication

`mediamtx.prod.yml` defines three users:

| User | Password | Can do |
|------|----------|--------|
| `publish` | `change-me` | Push streams |
| `any` | _(none)_ | Read and play streams |
| `admin` | `change-me-admin` | Access API |

Publishers authenticate in the URL: `rtmp://publish:change-me@server-ip:1935/neo`

Change both passwords before deploying.

## Docker networking note (Mac only)

Docker Desktop on macOS has a DNS resolution bug that prevents containers from reaching each other by service name. The override file assigns MediaMTX a static IP (`172.30.0.2`) on a dedicated bridge network as a workaround. This is not needed on Linux.

## Open firewall ports (production)

| Port | Protocol | Purpose |
|------|----------|---------|
| 1935 | TCP | RTMP publish |
| 8554 | TCP | RTSP |
| 8888 | TCP | HLS |
| 8889 | TCP | WebRTC signalling |
| 8189 | UDP | WebRTC media |

Keep port `9997` (API) private or restrict it by IP.

## Stop the stack

```bash
docker compose down
```
