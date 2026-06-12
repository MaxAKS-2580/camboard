# MERN Camera Dashboard with Kafka + WebRTC

This version uses WebRTC for live phone camera streaming.

## What changed

- Removed repeated JPEG frame uploads.
- Added WebRTC signaling APIs in the Express backend.
- The phone page uses `getUserMedia()` to access the camera.
- The dashboard receives the camera feed using `RTCPeerConnection`.
- Kafka is still used for camera status/connection events, not for raw video frames.

## Start Kafka

```bash
docker-compose up -d
```

If Kafka is not running, the app still works in fallback mode.

## Start backend

```bash
cd server
npm install
npm start
```

Backend runs on:

```txt
http://localhost:5000
```

## Start frontend

```bash
cd client
npm install
npm run dev
```

Vite is configured for normal HTTP on host `0.0.0.0` and port `5173`, so tunnel tools like Cloudflared, LocalTunnel, and ngrok can reach it without Bad Gateway errors.

## Open on laptop

```txt
http://localhost:5173
```

## Open on phone using a tunnel

Start a frontend tunnel and open the public HTTPS tunnel URL on your phone. For the phone camera page, use:

```txt
https://YOUR-FRONTEND-TUNNEL-URL/phone
```

Allow camera permission and tap **Start WebRTC Camera**.

## View the camera

Open the dashboard on laptop or through the tunnel:

```txt
http://localhost:5173
```

The Phone Camera tile should show the WebRTC feed.

## Notes

- Phone and laptop must be on the same Wi-Fi.
- Turn off VPN/WARP if connection fails.
- Allow Node/Vite through Windows Firewall.
- WebRTC is much better than REST uploads for live camera video.
- Kafka should be used for events/status, not raw video frames.

## Add public CCTV feeds by IP/URL

Open the dashboard and use **Add Public CCTV Feed**.

Examples of accepted values:

```txt
http://public-demo-server.example/mjpg/video.mjpg
http://192.168.1.50:8080/video
https://example.com/camera.jpg
```

The backend creates a camera tab and proxies the stream through:

```txt
/api/cameras/:cameraId/stream
```

Use only official public CCTV streams or cameras you own/have permission to access. Random exposed private IP cameras should not be used.

## Add IP cameras

Open the dashboard and use **Add IP Camera**.

Supported IP camera inputs:

```txt
rtsp://192.168.1.20:554/stream1
http://192.168.1.20:8080/video
http://192.168.1.20/snapshot.jpg
```

Fields:

- **URL**: `rtsp://`, `http://`, or `https://` camera stream URL.
- **Username / Password**: optional Basic Auth credentials for HTTP cameras; also injected into RTSP URLs for FFmpeg.
- **Stream kind**:
  - `Auto detect`
  - `RTSP via FFmpeg`
  - `HTTP MJPEG`
  - `HTTP Snapshot/Image`

RTSP streams are converted by the backend using FFmpeg and served as MJPEG through:

```txt
GET /api/cameras/:cameraId/stream
```

Install FFmpeg before using RTSP:

```bash
ffmpeg -version
```

If Windows says FFmpeg is not recognized, install FFmpeg and add it to PATH.

Use only cameras you own or are authorized to access.

## 1000-camera scaling notes

This build avoids opening every live stream at once. In **All Cameras** grid mode, only the first 16 tiles render live video; the rest render lightweight status cards. Open an individual camera tab to start/view that feed.

Recommended production architecture for 1000 cameras:

```txt
Cameras / RTSP / WebRTC / HTTP Snapshot
        ↓
Ingestion workers grouped by cameraId
        ↓
Kafka topics:
  camera-events
  camera-health
  camera-snapshots
        ↓
Dashboard API + WebSocket/SSE updates
        ↓
React dashboard with paginated/virtualized grid
```

Use Kafka for events, status, health, alerts, and snapshot metadata. Do not push raw 1000-camera video through Kafka.

## Bulk Import IP Cameras

The dashboard now supports bulk import of authorized IP cameras.

Frontend:
- Use **Bulk Import IP Cameras**.
- Paste one camera per line or upload a `.csv` / `.txt` file.

Supported row formats:

```txt
url
name,url
name,url,username,password,streamKind
```

Examples:

```txt
Gate Camera,rtsp://192.168.1.10:554/Streaming/Channels/101,admin,password,rtsp
Lobby Camera,http://192.168.1.11:8080/video,,,mjpeg
http://192.168.1.12/snapshot.jpg
```

Backend endpoint:

```http
POST /api/cameras/ip/bulk
```

Body:

```json
{
  "csvText": "Gate Camera,rtsp://192.168.1.10:554/Streaming/Channels/101,admin,password,rtsp",
  "defaultStreamKind": "auto"
}
```

For 1000-camera scale, the grid intentionally renders only the first 16 live feeds. The remaining cameras are lightweight tiles. Open an individual camera tab to stream that feed live.

## Bulk Import IP Cameras - URL Only

The bulk import box now accepts only camera URLs, one per line. No name, username, password, or stream type columns are required.

Examples:

```txt
rtsp://admin:password@192.168.1.10:554/Streaming/Channels/101
http://192.168.1.11:8080/video
http://192.168.1.12/snapshot.jpg
```

The backend auto-detects the camera type:
- `rtsp://...` → RTSP via FFmpeg
- `.jpg`, `.jpeg`, or `snapshot` URLs → snapshot/image
- other HTTP/HTTPS URLs → MJPEG-style stream

For authenticated cameras in bulk import, include credentials inside the URL, for example:

```txt
rtsp://admin:password@192.168.1.10:554/Streaming/Channels/101
```

## Cloudflare Tunnel / public URL note

If you tunnel the frontend only, the Vite dev server must proxy `/api` to the backend.
This ZIP includes that proxy in `client/vite.config.js`:

```js
proxy: {
  "/api": "http://localhost:5000"
}
```

Run these in separate terminals from this extracted ZIP:

```bash
cd server
npm install
npm start
```

```bash
cd client
npm install
npm run dev
```

Then tunnel the frontend port shown by Vite, for example:

```bash
cloudflared tunnel --url http://localhost:5173
```

If bulk import returns `API route not found`, restart the backend from this updated ZIP.

## Added: Analytics + 1000 Camera Scale Mode

This version includes:

- Per-camera analytics: status, uptime %, last seen, snapshots, refreshes, failures, stream requests, average response time.
- Dashboard analytics cards: total, online, offline, snapshots, refreshes, failures.
- Analytics APIs:
  - `GET /api/analytics`
  - `GET /api/analytics/:cameraId`
  - `POST /api/analytics/:cameraId/event`
- Bulk import is URL-only: paste one `rtsp://`, `http://`, or `https://` URL per line.
- Snapshot and refresh actions update analytics and Kafka events.
- 1000-camera mode keeps the All Cameras grid lightweight by rendering only the first 16 feeds live; open an individual tab to view any camera live.

### Tunnel setup

Run backend:

```bash
cd server
npm install
npm start
```

Run frontend:

```bash
cd client
npm install
npm run dev
```

Tunnel the frontend port:

```bash
cloudflared tunnel --url http://localhost:5173
```

The frontend uses Vite proxy for `/api` → `http://localhost:5000`, so keep backend running while using the frontend tunnel.

LocalTunnel alternative:

```bash
npx localtunnel --port 5173
```

ngrok alternative:

```bash
ngrok http 5173
```

### Bulk import format

```txt
rtsp://admin:password@192.168.1.10:554/Streaming/Channels/101
http://192.168.1.11:8080/video
http://192.168.1.12/snapshot.jpg
```


## Kafka fixed startup

This version uses `127.0.0.1:9092` instead of `localhost:9092` to avoid Windows/IPv6 connection issues.

Start Kafka from the project root:

```bash
docker rm -f camera-dashboard-kafka practical_lumiere
docker compose up -d kafka
```

Wait around 20 seconds, then start backend:

```bash
cd server
npm install
npm start
```

Check Kafka status:

```txt
http://localhost:5000/api/kafka/status
```

Reconnect manually if needed:

```bash
curl -X POST http://localhost:5000/api/kafka/reconnect
```

On Windows you can also double-click `reset-kafka.bat`.

## Real Vehicle Analytics Setup

This version removes simulated vehicle counts and analyzes actual captured frames from direct camera streams.

Supported for analytics:
- HTTP snapshot/image URLs
- HTTP MJPEG URLs
- RTSP IP camera URLs via FFmpeg

Not supported for analytics:
- Iframe/webpage feeds, because the backend cannot access the video pixels inside a third-party page.

### Install Python analytics dependencies

From the project root on Windows:

```bat
start-analytics-setup.bat
```

Or manually:

```bash
cd server/analytics_service
python -m pip install -r requirements.txt
```

For best detection quality, keep `ultralytics` installed. If YOLO cannot run, the app falls back to an approximate OpenCV contour detector.

### Use

1. Add a direct IP camera, MJPEG, snapshot, or RTSP URL.
2. Open that camera tab.
3. Click **Analyze Vehicles** or wait for the analytics panel to refresh.

The panel now shows actual frame analysis source:
- `YOLO vehicle detection`
- `OpenCV frame analysis`
- `Iframe cannot be analyzed`
- `Analyzer error`



## YOLO + EasyOCR only setup

This build does not use fake/simulated vehicle analytics and does not use the old OpenCV-only fallback for vehicle counting. Vehicle detection uses YOLO through `server/analytics_service/analyze_frame.py`; number plate OCR uses EasyOCR.

On Windows, run from the project root:

```bat
install-yolo-easyocr.bat
start-backend-and-analytics.bat
```

Do not create a `.venv` in the project root. The only Python environment used by analytics is:

```text
server\analytics_service\.venv
```

If VS Code points to `project-root\.venv`, delete that root `.venv` and select:

```text
server\analytics_service\.venv\Scripts\python.exe
```
#   c a m b o a r d  
 #   c a m b o a r d  
 #   c a m 1  
 