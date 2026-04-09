# USB Display: Nomad ↔ PC

Two goals:
1. **Mirror Nomad screen → PC** via USB
2. **Use Nomad as external display for PC** via USB

---

## Hardware Context

- SoC: Rockchip RK3566
- Display: E-ink, 1404×1872px, 16-level grayscale
- USB: USB-C 2.0 (no DisplayPort Alt Mode)
- OS: Chauvet (Android 11), ADB accessible via Settings > Security & Privacy
- Refresh: ~1–5 Hz full refresh; ~15 Hz A2 fast mode (ghosted)

---

## Goal 1: Nomad → PC (Screen Mirror over USB)

### What Already Works

```bash
scrcpy --video-codec=h264 --video-encoder=OMX.google.h264.encoder
```

Confirmed working on Chauvet firmware. The `--video-encoder` override is required — the RK3566 hardware H.264 encoder fails with scrcpy's MediaCodec surface setup; the software fallback (`OMX.google.h264.encoder`) works.

**How scrcpy works:**
- Pushes a Java server JAR to `/data/local/tmp/scrcpy-server.jar` on device
- Opens 3 ADB-tunneled TCP sockets: video (H.264), audio (OPUS), control (input events)
- Screen capture via Android `MediaCodec` API attached to SurfaceFlinger's compositor surface

### The Pen Stroke Problem

Pen strokes are invisible during drawing (~4 second lag until next full-screen refresh).

**Root cause:** Chauvet OS sends stylus input directly to the e-ink controller hardware, bypassing Android's SurfaceFlinger/compositor entirely. MediaCodec only sees what SurfaceFlinger composites. Pen strokes become visible only when the device triggers a full-screen compositor refresh (page turns, mode changes, etc.).

This is not fixable via scrcpy configuration — it is architectural.

**What mirrors fine:** page navigation, menus, typed text, completed pages.
**What doesn't:** real-time pen handwriting during drawing.

### Built-in MJPEG Stream (Wi-Fi Only)

The Nomad has a built-in screen sharing feature (swipe down → Screen Mirroring):

- Serves MJPEG at `http://<device-ip>:8080/screencast.mjpeg`
- Resolution: 2808×3744px (upscaled from physical 1404×1872)
- **Does capture pen strokes** — reads from e-ink frame buffer directly, not SurfaceFlinger
- Wi-Fi only — not accessible over USB
- ffmpeg cannot parse the stream (non-standard multipart boundaries); Puppeteer (headless Chrome) works as a workaround

**Research direction:** The MJPEG stream solves the pen stroke problem that scrcpy cannot. Bridging it to USB is a viable path:
- Identify what process serves port 8080 on the device (`adb shell ss -tlnp` or `adb shell netstat`)
- Investigate whether the stream can be forwarded over `adb forward tcp:8080 tcp:8080`
- Or replicate the frame buffer read in a custom app that streams over ADB instead

---

## Goal 2: PC → Nomad (External Display over USB)

### Why This Is Hard

Android has no built-in API to receive external display input. `VirtualDisplay` only outputs Android's own screen — it cannot inject an external video stream into the display pipeline. All approaches require a user-space app on the Nomad that decodes frames and renders them fullscreen via `SurfaceView`.

USB-C DisplayPort Alt Mode (sink) is a hardware requirement — the Nomad's USB-C 2.0 port does not support it. Dead end.

### Approach A: VNC via ADB Reverse Tunnel (Try First)

**Effort:** Low — no code needed
**Expected FPS:** 2–5 fps
**Status:** Likely works, untested on Nomad

**PC side:**
- Create a virtual display:
  - Linux: `xrandr --newmode` + dummy output, or `linux-display-extend` script
  - Windows: [virtual-display-rs](https://github.com/MolotovCherry/virtual-display-rs) IDD driver
- Run VNC server pointed at the virtual display only (TigerVNC / x11vnc / TightVNC)

**Nomad side:**
- Sideload AVNC (open source, no GMS required): `adb install avnc.apk`
- Connect: `adb reverse tcp:5900 tcp:5900` then point AVNC at `127.0.0.1:5900`

**Limitation:** VNC sends full-frame or delta JPEG/raw — fine for static content. E-ink refresh rate is the hard ceiling regardless.

### Approach B: spacedesk / SuperDisplay Sideload

**Effort:** Low
**Expected FPS:** 2–5 fps
**Status:** Unknown — GMS dependency risk on Chauvet OS

- spacedesk: installs a virtual display driver on Windows + Android APK receiver; USB mode uses `adb reverse tcp:28252 tcp:28252`
- SuperDisplay: similar architecture

**Risk:** Both apps may require Google Mobile Services (GMS) which is absent on the Nomad. Chauvet OS may also block the display overlay permissions these apps need. Worth testing before investing in custom development.

### Approach C: Custom H.264 Streaming App

**Effort:** High — requires building an Android app
**Expected FPS:** 5–10 fps
**Status:** Not built; technically sound

**PC side:**
- Capture virtual display region (ffmpeg `gdigrab` / `x11grab`)
- Encode H.264 with `ultrafast` preset
- Stream over TCP tunneled via `adb reverse tcp:9000 tcp:9000`

**Nomad side (Android app):**
- Open TCP socket on port 9000
- Receive H.264 NAL units → decode via `MediaCodec` (H.264 software decode works on all Android)
- Render decoded frames to fullscreen `SurfaceView`
- Send frames only on content change (dirty-rect diff) to respect e-ink refresh rate

**Key unknown:** Whether `MediaCodec` H.264 decode works in sideloaded apps on Chauvet OS. Different subsystem from encode — likely yes, but unconfirmed.

### Approach D: HDMI Capture Card + UVC

**Effort:** Medium + hardware cost
**Expected FPS:** 5–10 fps
**Status:** Technically possible, not ideal

- Plug a cheap USB HDMI capture card (MS2109 chip) into the Nomad via USB-C OTG
- PC outputs HDMI to capture card
- Nomad sees it as a UVC webcam — sideload a UVC camera app to display feed fullscreen
- Limitations: aspect ratio mismatch (1080p vs 1404×1872), added latency, not a true virtual monitor

---

## E-Ink Constraints (Apply to All Approaches)

| Constraint | Detail |
|---|---|
| Full refresh rate | ~1–5 Hz — physical limit, no software workaround |
| A2 fast mode | ~15 Hz but ghosted/low contrast |
| A2 from sideloaded app | Unknown — Boox has an unofficial API; Supernote equivalent unconfirmed; likely needs root |
| Pen rendering | Bypasses Android compositor — creates visual conflicts if displaying PC content while using stylus |
| GMS absent | Rules out apps with hard Google Play Services dependencies |

---

## Research Phases

### Phase 1 — Validate No-Code Approaches
- [ ] Confirm scrcpy works on current firmware
- [ ] Test `adb forward tcp:8080 tcp:8080` — does the MJPEG stream become accessible on PC?
- [ ] Sideload AVNC + set up virtual display + VNC on PC
- [ ] Test spacedesk APK sideload — does Chauvet OS grant required permissions?

### Phase 2 — Bridge MJPEG to USB
- [ ] `adb shell ss -tlnp | grep 8080` — identify the process serving the stream
- [ ] Dump the APK of Supernote's screen mirroring feature for analysis
- [ ] Determine if stream can be forwarded or if a custom relay is needed

### Phase 3 — Custom Streaming App
- [ ] Build minimal Android APK: TCP socket → MediaCodec decode → SurfaceView
- [ ] PC-side: ffmpeg virtual display capture → H.264 → TCP over ADB reverse
- [ ] Validate MediaCodec decode works on Chauvet OS

### Phase 4 — A2 Fast Mode (Requires Root)
- [ ] Study Boox's undocumented e-ink refresh waveform API as template
- [ ] After rooting, investigate `/sys/` entries for e-ink controller on RK3566
- [ ] Triggering A2 mode could push effective FPS to ~15 — viable for text/code display

---

## Reference Projects

| Project | Relevance |
|---|---|
| [Genymobile/scrcpy](https://github.com/Genymobile/scrcpy) | Goal 1 — works today with encoder override |
| [borzunov/remoteink](https://github.com/borzunov/remoteink) | Goal 2 — PocketBook (Linux e-ink) as PC monitor via custom TCP |
| [USKhokhar/linux-display-extend](https://github.com/USKhokhar/linux-display-extend) | Goal 2 — xrandr + x11vnc virtual display approach |
| [MolotovCherry/virtual-display-rs](https://github.com/MolotovCherry/virtual-display-rs) | Goal 2 — Windows IDD virtual display driver |
| [Prayag2/android-as-monitor-linux](https://github.com/Prayag2/android-as-monitor-linux) | Goal 2 — Android as monitor, Linux |
| [Modos-Labs/Glider](https://github.com/Modos-Labs/Glider) | Goal 2 — open-source e-ink HDMI monitor (custom hardware, not Android) |
| scrcpy issue [#5788](https://github.com/Genymobile/scrcpy/issues/5788) | Nomad-specific scrcpy findings |
| [spacedesk](https://spacedesk.net/) | Goal 2 — commercial app, worth testing via sideload |
| [SuperDisplay](https://superdisplay.app/) | Goal 2 — commercial app, worth testing via sideload |
