# Reel API

Send a business description and a few of the business's own photos.
Get back **one** vertical video (9:16) for Instagram, TikTok or Reels.

**Important: you get ONE finished video, not one video per photo.**

Each photo is turned into a short moving clip, and all the clips are joined
together into a single video file. The individual clips are never returned.
Three photos and a 12 second reel means three 4-second clips inside one
12-second video.

All the text on screen is drawn by this service, not by the video model, so the
business name is always spelled correctly.

The video has no sound. Seedance 1.0 has no audio setting.

---

## Setup

```bash
cd reel_create_api
pip install -r requirements.txt
cp .env.example .env          # then put your keys in it
```

You need two keys: `FAL_KEY` for the video model, and either
`ANTHROPIC_API_KEY` or `OPENAI_API_KEY` for the writing. Set
`REEL_LLM_PROVIDER` to say which one to use.

## Run it

```bash
uvicorn app:app --port 8400
```

---

## 1. Create a reel

```bash
curl -X POST http://127.0.0.1:8400/reel/create \
  -H "Content-Type: application/json" \
  -d '{
    "business_name": "Legacy Cooling",
    "business_description": "Family-owned HVAC company serving Greater Austin for over 20 years. Air conditioning, heating repair, emergency service and indoor air quality.",
    "image_urls": [
      "https://example.com/photo1.jpg",
      "https://example.com/photo2.jpg",
      "https://example.com/photo3.jpg"
    ],
    "resolution": "480p",
    "duration": 12
  }'
```

It answers straight away. It does **not** wait for the video:

```json
{"request_id": "918f1df6aa0f40bd9972abc7a6bd7441",
 "status": "queued",
 "poll": "/reel/918f1df6aa0f40bd9972abc7a6bd7441"}
```

A reel takes about 4 to 6 minutes, which is too long to hold an HTTP request
open, so you poll for it.

### Request fields

| Field | Required | Default | Notes |
|---|---|---|---|
| `business_name` | yes | — | 2 to 200 characters |
| `business_description` | yes | — | 40 to 12000 characters |
| `image_urls` | yes | — | 1 to 4 photos. An `https://` link or a `data:image/jpeg;base64,...` string |
| `duration` | no | `12` | 6 to 24 seconds, for the whole reel |
| `resolution` | no | `1080p` | `480p`, `720p` or `1080p` |
| `website` | no | — | printed on the end card |
| `cta` | no | — | your own call to action, e.g. `CALL TODAY` |
| `logo_url` | no | — | drawn on the end card |
| `model_tier` | no | `pro` | `pro` or `pro_fast` (cheaper, lower quality) |
| `format`, `style`, `ending` | no | `auto` | the service picks per business so two reels do not look the same |
| `review` | no | `off` | `off`, `once`, `twice`. Costs extra |

Anything not listed is ignored. Bad values return `{"error": "..."}` with a
400 status.

## 2. Check if it is ready

```bash
curl http://127.0.0.1:8400/reel/918f1df6aa0f40bd9972abc7a6bd7441
```

While it works you get `"status": "running"` and a line saying what it is doing
(`"animating photo 2 of 3"`).

When it is done you get `"status": "completed"` and the full result:

```json
{
  "status": "completed",
  "video_url": "/reel/918f1df6aa0f40bd9972abc7a6bd7441/video",
  "video": {"duration_s": 12.0, "size": [480, 854], "bytes": 3119950},
  "cost": {"total_usd": 0.4204, "by_stage": {...}},
  "clips": [...],
  "copy": {...}
}
```

`video_url` is the **one** finished reel. That is the only video the API gives
you.

The `clips` list is information only: the prompt written for each photo, how
long that part lasts, and what it cost. It holds **no video links**. The
separate clip files are kept on the server as working files, and are not served.

## 3. Download the video

```bash
curl -o reel.mp4 http://127.0.0.1:8400/reel/918f1df6aa0f40bd9972abc7a6bd7441/video
```

One MP4 file.

## Redraw without paying again

```bash
curl -X POST http://127.0.0.1:8400/reel/<id>/rerender -d '{}'
```

This draws the text and the layout again using the clips that were already
made. The video model is not called, so it costs about half a cent instead of
a full reel. Use it to fix wording or change the format.

## Other endpoints

| Endpoint | What it gives |
|---|---|
| `GET /limits` | photo count, durations, resolutions |
| `GET /formats` | the reel formats and what each one is for |
| `GET /styles` | the visual styles and endings |
| `GET /queue` | jobs running now |
| `GET /health` | `{"ok": true}` |

---

## Limits

- **1 to 4 photos.** More than 4 is refused. Three is what we normally send.
- **6 to 24 seconds** total. The time is split between the photos.
- **9:16 only** (vertical).
- **No audio.**

## Cost

Costs are calculated, not guessed. The video figure comes from fal's published
token formula, the writing figures from the token counts the language API
returns. Every job reports its own cost under `cost`, broken down by stage.

Video cost for a 12-second reel made of three clips:

| Resolution | Video cost |
|---|---|
| 480p | $0.29 |
| 720p | $0.65 |
| 1080p | $1.46 |

The writing adds about $0.13. Five reels run end to end at 480p on
15 Oct 2026 cost **$0.42 each on average**. The cost could be more.

These are computed figures. Check them against fal.ai/dashboard/usage before
you bill anyone.
