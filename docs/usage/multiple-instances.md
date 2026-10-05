# Running Multiple Instances (Multi-Camera)

**Yes — you can run several instances with several cameras on one machine.** Each
instance is a separate process with its own port, its own model/prompt settings,
and its own transcript file.

## Why one instance per camera

A server process holds **one** VLM service. Everything downstream is shared:

- one "current response" that the UI displays
- one inference slot — while a frame is being analysed, frames from any other
  source are dropped (`VLM busy, skipping frame`)
- one prompt and one model at a time

So pointing two cameras at a single instance makes them compete for one slot and
interleaves their descriptions in the UI. For genuinely independent cameras,
run one instance per camera.

## Starting several instances

Give each one a distinct port:

```bash
cd /path/to/live-vlm-webui

.venv/bin/live-vlm-webui --port 8090 &   # camera 1
.venv/bin/live-vlm-webui --port 8091 &   # camera 2
.venv/bin/live-vlm-webui --port 8092 &   # camera 3
```

Or let each one find its own free port:

```bash
for i in 1 2 3; do .venv/bin/live-vlm-webui --port 8090 --auto-port & done
```

`--auto-port` scans up to 10 ports from the one given. Without it, a taken port
fails fast with the PID holding it:

```
❌ Port 8090 is already in use by PID 1717096 (python)
   Pick another with --port <N>, or add --auto-port to
   find the next free port automatically
```

## Assigning cameras

Cameras are picked **in the browser**, not on the server — the video is captured
by the page and streamed over WebRTC. So:

1. Open `https://localhost:8090` → choose *Camera 1* in **Camera Selection** → Start
2. Open `https://localhost:8091` → choose *Camera 2* → Start
3. …and so on

Each tab drives its own instance. Give each one its own prompt if you want
different analysis per camera.

## Transcripts stay separate

Each instance claims its own transcript file atomically at startup, so instances
started in the same second never share one:

```
~/.local/share/live-vlm-webui/transcripts/
├── session-20251217-142530.jsonl      ← instance A
├── session-20251217-142530-1.jsonl    ← instance B (same second)
└── session-20251217-142530-2.jsonl    ← instance C
```

Sequence numbers stay gapless per file, and pausing recording on one instance
does not affect the others.

To keep each camera's transcript obviously identifiable, give each an explicit
directory or file:

```bash
.venv/bin/live-vlm-webui --port 8090 --transcript-dir ~/vlm-logs/front-door &
.venv/bin/live-vlm-webui --port 8091 --transcript-dir ~/vlm-logs/garage &
```

Every record also carries a `source` field (`webcam`, `rtsp:<session-id>`), so a
merged view can still be demultiplexed:

```bash
jq -r 'select(.kind=="analysis") | "\(.ts) [\(.source)] \(.response)"' \
   ~/vlm-logs/*/session-*.jsonl | sort
```

## Multiple RTSP streams in one instance

The RTSP API accepts a `session_id`, so one instance *can* hold several streams:

```bash
curl -k -X POST https://localhost:8090/api/rtsp/start \
  -H 'Content-Type: application/json' \
  -d '{"rtsp_url": "rtsp://...", "session_id": "front-door"}'
```

They still share the single VLM slot, so throughput is divided between them and
the UI shows whichever finished last. The transcript does tag each record with
`source: rtsp:front-door`, so the *data* stays separable even though the live
view does not. Prefer one instance per stream unless you deliberately want
round-robin sampling across cameras.

## What actually limits you

| Resource | Per instance | Notes |
|----------|-------------|-------|
| GPU / VRAM | none | Inference happens in your VLM backend (Ollama, vLLM…), not in the server |
| **VLM backend concurrency** | **the real limit** | N instances issue N concurrent requests; a backend serving one at a time serialises them and latency scales with N |
| CPU | modest | Frame colour conversion, plus an NVML poll every 0.25 s per instance |
| Ports | 1 | Plus WebRTC UDP ports negotiated per connection |

If descriptions start lagging with several cameras, the bottleneck is almost
always the backend, not the servers. Raise **Frame Processing Interval** per
instance (process every Nth frame) to reduce request rate, or run a backend
configured for concurrent requests (vLLM handles this far better than Ollama).

## Notes

- **SSL certificates** are shared from the config directory. They are generated
  on first run; if several instances cold-start simultaneously on a machine that
  has never generated them, they can race. Start one instance first (or run
  `./scripts/generate_cert.sh`) so the certs exist, then start the rest.
- **`live-vlm-webui-stop` stops every instance**, not just one. To stop a single
  instance, kill its PID:
  ```bash
  kill $(ss -tlnp | grep ':8091 ' | grep -oP 'pid=\K[0-9]+')
  ```
