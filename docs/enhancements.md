# Enhancement: Youtube2Webpage Web Application

## Overview

Convert the existing CLI utility into a web application where a user submits a YouTube URL and receives a link to the generated transcript page served on the same domain.

This requires server-side processing — `yt-dlp` and `ffmpeg` cannot run in a browser or serverless environment. The existing `main_video_handling.js` has the core logic and will be adapted (one Electron import must be removed).

---

## Infrastructure Recommendation

**Hetzner CX22 — ~$4.15/month**

| Provider | Plan | Price/mo | RAM | Storage | Verdict |
|---|---|---|---|---|---|
| **Hetzner** | **CX22** | **~$4.15** | **4 GB** | **40 GB SSD** | **Best value** |
| DigitalOcean | Basic 2GB | $12 | 2 GB | 50 GB | 3× more expensive |
| DigitalOcean | Basic 1GB | $6 | 1 GB | 25 GB | RAM too tight for ffmpeg |
| Fly.io | shared-cpu-1x | $1.94 + storage | 256 MB | Expensive volumes | Insufficient RAM |
| Render | Starter | $7 | 512 MB | No persistent disk | No persistent filesystem |

Setup: Ubuntu 22.04 LTS + `ffmpeg` (apt) + `yt-dlp` (pip) + Node.js + `pm2` for process management.

---

## File Structure

```
/
├── server/
│   ├── package.json          # express, uuid
│   ├── server.js             # Express routes
│   ├── jobRunner.js          # In-process job queue
│   ├── videoProcessor.js     # Core logic (adapted from main_video_handling.js)
│   ├── public/
│   │   └── index.html        # Submission form with JS polling
│   └── pages/                # Created at runtime — one subdir per job
│       └── {jobId}/
│           ├── index.html    # Generated transcript page
│           ├── styles.css    # Copied from repo root styles.css
│           └── images/       # JPEG screenshots from ffmpeg
│               └── *.jpg
├── docs/
│   └── enhancements.md       # This file
└── ... (existing files untouched)
```

The Electron app in `Youtube2Webpage/` is left untouched.

---

## Implementation Steps

### 1. `server/package.json`

Minimal dependencies — no build step, no TypeScript, no framework beyond Express.

```json
{
  "name": "youtube2webpage-server",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "express": "^4.18.2",
    "uuid": "^9.0.0"
  }
}
```

### 2. `server/videoProcessor.js`

Adapted from `Youtube2Webpage/src/main_video_handling.js`:

- **Remove**: `import { nativeImage } from "electron"` — the only line that blocks reuse
- **Copy verbatim**: `parseVTT()`, `convertTimestampToSeconds()`, `yt-dlp` args from `downloadVideo()`
- **Replace**: `getThumbnail()` (pipes to Electron's nativeImage → base64 data URL) with `generateImages()` that writes `.jpg` files directly to disk, matching the Perl script's ffmpeg pattern
- **Port**: HTML generation template from `yt-to-webpage.pl` (~20 lines of `<ul>/<li>` markup)
- **Add**: `processJob(jobId, url, outputDir)` — top-level orchestrator
- **Fix**: No `process.chdir()` — use absolute paths and `--paths` arg to `yt-dlp` throughout
- **Cleanup**: Delete the downloaded video file after screenshots are extracted (`.webm` can be 500 MB+)
- **Error handling**: If no `.vtt` file found after download, throw a user-friendly error (some videos have no captions)

VTT file discovery (language code varies per video):
```js
const vttFile = fs.readdirSync(workDir).find(f => f.endsWith('.vtt'));
```

### 3. `server/jobRunner.js`

Simple in-memory Map + serial queue. One job runs at a time — ffmpeg is CPU-heavy and concurrency provides no benefit on a single server.

- `enqueue(jobId, url)` — adds job to queue, triggers drain
- `getJob(jobId)` — returns current job state
- Job states: `queued → processing → done | error`
- Wrap `processJob` in a `Promise` so the event loop stays unblocked during polling

### 4. `server/server.js`

```
POST /jobs      → validate URL, enqueue, return { jobId, statusUrl }
GET  /jobs/:id  → return { id, status, pageUrl? (when done), error? (when failed) }
GET  /pages/*   → express.static serving generated transcript pages
GET  /          → serve public/index.html
```

URL validation: `^https://(www\.)?youtube\.com/watch`

### 5. `server/public/index.html`

Single static page — no framework:
- URL `<input>` + submit button
- On submit: `fetch('POST /jobs')` → get `jobId`
- Poll `GET /jobs/:id` every 3 seconds
- On `status === 'done'`: render a clickable link to `pageUrl`
- On `status === 'error'`: show the error message

3-second polling is chosen over WebSockets — negligible server load, zero extra dependencies, and the latency is imperceptible relative to a multi-minute processing job.

---

## Request / Response Flow

```
User                    Server                        Disk
 |                         |                            |
 |  POST /jobs {url}       |                            |
 |-----------------------> |                            |
 |  202 { jobId }          |  enqueue(jobId, url)       |
 | <---------------------- |                            |
 |                         |  yt-dlp downloads video    |
 |  GET /jobs/:id          |  + subtitles (.vtt)        |
 |-----------------------> |                            |
 |  { status: "processing"}|  ffmpeg extracts frames    |
 | <---------------------- |                            |
 |  GET /jobs/:id (3s)     |  HTML generated            |
 |-----------------------> |  video file deleted        |
 |  { status: "done",      |                            |
 |    pageUrl: "/pages/…" }|            pages/{jobId}/  |
 | <---------------------- |           index.html  <----|
 |                         |           styles.css  <----|
 |  GET /pages/{jobId}/    |           images/*.jpg <---|
 |-----------------------> |                            |
 |  Generated HTML page    |                            |
 | <---------------------- |                            |
```

---

## Source Files Referenced

| Existing file | How it is used |
|---|---|
| `Youtube2Webpage/src/main_video_handling.js` | Copy `parseVTT()`, `convertTimestampToSeconds()`, yt-dlp args |
| `yt-to-webpage.pl` | Port ffmpeg invocation (disk-write mode) and HTML template |
| `styles.css` | Copied into each job's output directory |
| `example/index.html` | Reference for exact output HTML structure |

---

## Verification Checklist

- [ ] `cd server && npm install && node server.js` starts without errors
- [ ] Submit a YouTube URL → status updates appear every 3 seconds
- [ ] Final link loads the transcript page with screenshots and captions
- [ ] Images load correctly (relative paths from `images/` subdir)
- [ ] Caption deeplinks back to YouTube at the correct timestamp
- [ ] Processing a second URL while one is queued works correctly
- [ ] Videos with no captions return a clear error message
