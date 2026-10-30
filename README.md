This is a [Next.js](https://nextjs.org) project bootstrapped with [`create-next-app`](https://nextjs.org/docs/app/api-reference/cli/create-next-app).

## Getting Started

First, run the development server:

```bash
npm run dev
# or
yarn dev
# or
pnpm dev
# or
bun dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

For a live broadcast, use the custom production server rather than the development server. Build before the event, then start the app with:

```powershell
npm run build
npm run start
```

The production start command keeps the app's Socket.IO and RTSP server enabled. Run the build and startup check before the broadcast day; do not rebuild while live.

## Connect RTSP cameras

The browser cannot play RTSP directly. After updating the project, run `npm install`, start Docker Desktop, then run `docker compose up -d` from the project folder. This starts the included MediaMTX gateway. In **Hub → Bảng Cài Đặt Đường Truyền & Tên Cam**, enter the camera's actual RTSP/RTSPS URL and choose **Kết nối RTSP**.

Set up each camera slot with its working RTSP URL and credentials first. In the settings tab, **Ghim serial camera đã cấu hình** reads ONVIF `GetDeviceInformation` from every saved RTSP IP and automatically binds each unique serial to that same camera slot. Serial-to-slot bindings are stored in `data/onvif-camera-bindings.json`; RTSP paths and credentials remain in `data/rtsp-sources.json`. When an IP is entered manually and its RTSP stream connects, the app also attempts to read and save that camera's ONVIF serial automatically. On a new network, **Đồng bộ thiết bị** tries WS-Discovery and scans the computer's directly connected private IPv4 subnet (up to 1,022 hosts) on every synchronization so newly added cameras are not missed even if multicast discovery does not return them. It uses saved RTSP credentials by default, or the credentials currently entered in the UI, to read serials. When a serial matches a binding, the app automatically configures and connects that camera using its saved RTSP path/account at the newly found IP; it also reconnects a bound camera if its current MediaMTX path is not ready, but leaves a healthy stream running. The results list shows configured/bound cameras plus newly discovered devices with a unique ONVIF serial as **camera mới, chưa gán**; devices with no readable serial and duplicate unassigned serials are omitted to avoid ambiguous selection. Select a new candidate IP in the intended CAM row and press **Kết nối** once to assign it; after the RTSP stream is accepted, the app attempts to save that camera's serial automatically so subsequent synchronization can auto-connect it. Each camera settings card reports whether its stream, serial, and preview-confirmed IP are saved. If a camera has no discovered IP, enter its private IPv4 address in **Hoặc nhập IP camera** and press **Kết nối**; for an already configured camera the saved RTSP path/account are retained, and a new camera uses the standard Dahua/Imou H.264 path. New cameras automatically reuse the shared credentials from an existing saved RTSP source when the credential fields are blank; otherwise the credentials entered above are used. This requires the cameras and app computer to be on reachable local networks, ONVIF enabled, and credentials accepted by ONVIF; routing/VLAN/firewall rules can still block discovery or the stream. Cameras that do not provide a unique serial cannot be auto-matched. An IP is hidden from other camera selectors as soon as it has a configured RTSP source, preventing two camera slots from using the same address; the server also rejects duplicate assignments. Verified IP assignments are saved locally in `data/verified-camera-ips.json` after preview playback succeeds. Use **Gỡ IP** on a configured camera to stop its RTSP source, clear its saved assignment, and return the IP to the other camera selectors. Cameras using a different RTSP path or without ONVIF discovery support still need custom RTSP configuration.

The RTSP URL must use the real camera IP and credentials; replace all example placeholders. Only cameras in the selected Program and Preview scenes are transcoded by FFmpeg to H.264 Baseline, 1280×720, 30 fps, and sent to MediaMTX for browser/OBS playback. At startup, the app checks the available encoders and limits camera encoding to six NVIDIA NVENC sessions, reserving GPU capacity for OBS; remaining feeds use Intel Quick Sync when available, then software `libx264` as a fallback. This avoids exceeding the NVIDIA hardware-session limit on multi-camera systems while keeping CPU use low. The app includes a Windows/Linux/macOS FFmpeg binary through `ffmpeg-static`; set `FFMPEG_PATH` to use another FFmpeg executable. Restart the app for encoder changes to take effect. Defaults are:

- Control API: `http://127.0.0.1:9997`
- RTSP ingest: `rtsp://127.0.0.1:8554`
- WebRTC/WHEP: `http://localhost:8889`

The app remembers the exact camera list (including cameras added or removed), display names/numbers, Preview and Program selections, layout, broadcast settings, and tournament-table assignments in `data/camera-display-settings.json`. Camera source URLs and credentials are stored separately in `data/rtsp-sources.json`; the active stream URL is recreated when the server starts.

## Connect DroidCam and Action Cam

In **Hub → Nguồn camera ngoài**, select the destination CAM. For an Android DroidCam, keep the phone and app server on the same reachable LAN, open DroidCam on the phone, then choose **Dò DroidCam** and connect a discovered device. Discovery checks private IPv4 subnets of up to 4,094 hosts for the DroidCam HTTP endpoint on port `4747`; on larger subnets it checks the /22 block containing the server. Provide both username and password if authentication is enabled. The source is transcoded by FFmpeg into the same MediaMTX/WebRTC output as RTSP cameras.

For an Action Cam, enter its private IPv4 address and RTSP credentials, then connect it separately through the RTSP camera path. Changing a CAM slot from DroidCam to Action Cam creates a fresh RTSP source rather than reusing the DroidCam HTTP URL. DroidCam sources are not included in ONVIF serial discovery/binding.

If MediaMTX uses different addresses, set `MEDIAMTX_API_URL` and `MEDIAMTX_WHEP_URL` before starting the app. `MEDIAMTX_WHEP_URL` is the base URL reachable by browsers; for example, on a LAN it may be `http://192.168.1.20:8889`. Allow MediaMTX's WebRTC signaling and media ports through the firewall.

For Dahua cameras, **Kết nối RTSP** uses the supplied stream. The server transcodes H.265 and other supported camera codecs to H.264 before OBS receives the output.

For a sharper Dahua picture, adjust the camera's stream resolution and bitrate in the camera's own settings. Use H.264 if OBS Browser Source does not support the selected codec. The fullscreen camera view reports the received video resolution. Auto-Cycle randomly chooses among configured cameras at the selected interval, prepares the next stream, and switches only after it becomes ready.

Manual scene changes are applied to Output as soon as the Hub state arrives; a camera that is still connecting or unavailable does not hold the previous scene on screen. Its tile shows the stream's connection status while other cameras in the new scene remain visible.

Keep the Output page connected while switching Program scenes. Open `http://localhost:3000/output` in a browser during local testing, or add `http://localhost:3000/output` as an OBS Browser Source (replace `localhost:3000` with the app server address when OBS runs on another machine). The Hub provides **Mở Output** for convenience. Without an Output page connected, scene cuts are intentionally held so a camera is not switched live before Output has received its first frame.

The browser output automatically retries camera streams after connection failures, prolonged WebRTC disconnections, or a video feed that stops advancing. The server monitors FFmpeg encoding progress and restarts a transcoder that exits or stops producing progress; it waits up to 30 seconds for MediaMTX to confirm a newly configured feed. This reduces recovery time but cannot guarantee uninterrupted video: camera power, RTSP availability, LAN quality, firewall rules, browser limits, and available CPU/bandwidth still matter. For best stability, use wired networking, keep MediaMTX running, allow WebRTC UDP port `8189`, and avoid enabling more camera feeds than the output machine can decode.

Camera audio is forwarded as Opus when the source provides an audio track. In the Hub's **Program** panel, use the audio button on each camera tile to choose which camera audio is heard on the Output. Audio is off for all cameras by default, and Preview-only cameras remain muted.

Only camera streams in Program and the selected Preview scene are transcoded. Selecting a new Preview scene starts and verifies those incoming streams while the current Program stays live. When **Cut To Scene** is pressed, the server asks the OBS Output page to open the incoming WebRTC feeds invisibly, waits for actual video playback and a minimum 10-second preparation period, then switches Program and stops feeds no longer in use. If Output does not confirm every incoming image, the current Program stays live and the Hub reports the failure. Saved RTSP URLs, credentials, ONVIF serial bindings, and verified IPs remain saved while their streams are idle. The Output page and Hub camera tiles subscribe only to cameras in the current Program or Preview, plus the temporary staged scene during the cut, so inactive cameras do not consume decoder resources. Auto-cycle starts and verifies the next camera before switching, then stops the previous feed. The Output multiview automatically chooses a rows/columns arrangement that maximizes the combined visible area of the 16:9 camera images for the current browser window. It centers an incomplete final row and keeps the full camera frame visible rather than cropping its edges. For a larger audience view, use a larger Output/OBS canvas or select fewer cameras in the Program scene.

### Long live-session checks

For a multi-hour broadcast on Windows, keep the laptop plugged into power, select a performance power mode, and set Windows sleep/hibernation to stay off during the broadcast. Prefer wired Ethernet for the camera network and the OBS/output machine where possible. In OBS, use the NVIDIA NVENC H.264 encoder when available, then test the complete setup with the planned camera count and scene layout before going live.

The project includes a read-only health monitor for a camera soak test. With the production app and MediaMTX running, open a separate PowerShell window in the project folder and run:

```powershell
.\scripts\monitor-live.ps1 -CameraCount 18
```

It samples MediaMTX path readiness, FFmpeg process count, estimated FFmpeg CPU usage, and FFmpeg memory every five seconds, and writes a CSV under `data`. Leave it running during a 20–30 minute full-load test and review the CSV for missing camera paths or increasing resource use. It does not change or restart streams. The CPU estimate covers FFmpeg processes only, not the browser, OBS, or total system load; use Task Manager and OBS Stats for those. Press Ctrl+C to stop the monitor.

If the Hub reports HTTP 401/403, MediaMTX requires Control API authentication. Add an internal user with `action: api` to `mediamtx.yml`, then start the app with matching `MEDIAMTX_API_USERNAME` and `MEDIAMTX_API_PASSWORD` environment variables. For example, in PowerShell before `npm run dev`:

```powershell
$env:MEDIAMTX_API_USERNAME = "your-api-user"
$env:MEDIAMTX_API_PASSWORD = "your-api-password"
npm run dev
```

Do not add API credentials to source control. The user needs the `api` permission in MediaMTX; camera RTSP credentials are separate.

Keep the MediaMTX Control API and ingest private and do not expose them to the public internet. The included config grants API, camera publishing, and camera viewing permission to local requests without a password, while Docker binds ports `9997` and `8554` only to `127.0.0.1`. Configured RTSP and DroidCam source URLs are saved locally in the ignored `data/rtsp-sources.json` file and restored to MediaMTX when the app starts. This file can contain camera credentials in plain text; keep the project folder private. Credentials are not included in the broadcast state sent to viewers. URL-encode special characters in RTSP usernames and passwords.

You can start editing the page by modifying `app/page.tsx`. The page auto-updates as you edit the file.

This project uses [`next/font`](https://nextjs.org/docs/app/building-your-application/optimizing/fonts) to automatically optimize and load [Geist](https://vercel.com/font), a new font family for Vercel.

## Learn More

To learn more about Next.js, take a look at the following resources:

- [Next.js Documentation](https://nextjs.org/docs) - learn about Next.js features and API.
- [Learn Next.js](https://nextjs.org/learn) - an interactive Next.js tutorial.

You can check out [the Next.js GitHub repository](https://github.com/vercel/next.js) - your feedback and contributions are welcome!

## Deploy on Vercel

The easiest way to deploy your Next.js app is to use the [Vercel Platform](https://vercel.com/new?utm_medium=default-template&filter=next.js&utm_source=create-next-app&utm_campaign=create-next-app-readme) from the creators of Next.js.

Check out our [Next.js deployment documentation](https://nextjs.org/docs/app/building-your-application/deploying) for more details.
