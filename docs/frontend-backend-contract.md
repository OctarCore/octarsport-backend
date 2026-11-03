# OctarSport: Frontend ↔ Backend API Contract

| | |
|---|---|
| **Status** | Draft v0.1 |
| **Owners** | Adrian (backend), Yura (frontend) |
| **Last updated** | 2026-11-03 |
| **Related** | Backend ↔ AI contract (separate document, owned by Adrian and Artur) |

This document is the single source of truth for how the web client talks to the backend.
Any change goes through a Pull Request that both owners approve.

Markers used below:

- **[MVP]**: must exist in the first working version.
- **[Planned]**: agreed direction, implemented after the MVP. The frontend should not depend on it yet.
- **[Draft]**: a value or decision that is a proposal and is open for discussion (see section 12).

## Table of contents

1. Purpose and scope
2. Environments and general conventions
3. Domain concepts
4. Endpoints
5. Data schemas
6. Errors
7. Limits and validation
8. Frontend implementation guide
9. TypeScript types
10. Mocking and dev workflow
11. Versioning and change process
12. Open questions
13. Readiness checklist

---

## 1. Purpose and scope

OctarSport analyses sports technique from video. The user uploads a video, the backend processes it
asynchronously with the AI service, and the frontend shows the video with a skeleton overlay,
technical errors, a score and recommendations.

The frontend talks **only to the backend**. It never calls the AI service directly.

Out of scope for the MVP: authentication, user accounts, payments, real-time push (WebSocket/SignalR).

## 2. Environments and general conventions

### 2.1 Base URLs

| Environment | Backend base URL | Frontend origin (CORS) |
|---|---|---|
| Local dev | `http://localhost:8080` | `http://localhost:5173` |
| Docker Compose | `http://localhost:8080` | `http://localhost:5173` |
| Production | TBD | TBD |

- Swagger UI (generated from the real implementation): `GET /swagger`
- OpenAPI document: `GET /swagger/v1/swagger.json`
- The frontend reads the base URL from the environment variable `VITE_API_BASE_URL`.
  It must never be hardcoded in components.

### 2.2 Format rules

- All request and response bodies are **JSON, UTF-8**, except the video upload (`multipart/form-data`)
  and the video download (binary).
- Field names are **`snake_case`**.
- Identifiers (`analysis_id`) are **UUID v4 strings**.
- Timestamps are **ISO 8601 in UTC**, for example `2026-11-03T12:00:00Z`.
- Durations are in **seconds** (`*_sec`), floating point.
- Coordinates are **normalized floats** (see section 5.2).
- Unknown fields must be **ignored** by the client. The backend may add new fields without a new version.
- A field that is allowed to be absent is always present with `null`, it is never omitted.
  This keeps TypeScript types simple.

### 2.3 Headers

| Header | Direction | Description |
|---|---|---|
| `Content-Type` | request | `multipart/form-data` for upload, `application/json` otherwise |
| `Accept` | request | `application/json` (video endpoint: `video/*`) |
| `X-Request-Id` | response | Unique id of the request. The frontend should log it and show it in error screens so problems can be traced in backend logs |
| `Content-Encoding` | response | `gzip` for large JSON (landmarks). Browsers decompress it automatically |
| `Location` | response | Set on `202 Accepted` after upload, points to the status endpoint |
| `Retry-After` | response | Set on `429` and `503`, seconds to wait |

### 2.4 CORS

The backend allows the frontend origin from the table above, methods `GET, POST, DELETE, OPTIONS`,
and exposes the headers `Location`, `X-Request-Id`, `Retry-After`, `Content-Range`, `Accept-Ranges`.
Cookies and credentials are **not** used in the MVP.

### 2.5 Authentication

None in the MVP. All endpoints are public. Consequently, anyone who knows an `analysis_id` can read that
analysis, which is why ids are unguessable UUIDs. Authentication is a post-MVP topic and will be added
through an `Authorization: Bearer <token>` header without changing the paths.

### 2.6 Paths

All business endpoints live under `/api`. There is no version in the path in v1. A breaking change
will introduce `/api/v2` (see section 11).

## 3. Domain concepts

### 3.1 Analysis

An **analysis** is one uploaded video plus everything produced from it.

### 3.2 Lifecycle and statuses

```
            upload accepted               worker picks it up            AI finished
 (none) ───────────────────► queued ───────────────────► processing ───────────────► completed
                                                            │
                                                            └─────────────────────► failed
```

| Status | Meaning | Frontend behavior |
|---|---|---|
| `queued` | Video stored, waiting for a free worker | Show "In queue", keep polling |
| `processing` | AI service is analysing the video | Show spinner, keep polling |
| `completed` | Results are ready | Stop polling, load results and landmarks |
| `failed` | Processing failed, `error` is filled | Stop polling, show the error and a "Try again" (upload again) button |

`completed` and `failed` are **terminal**. A status never goes back.

### 3.3 Sports and movements [Draft]

`sport` and `movement` are open string values (lowercase `snake_case`), so new sports can be added
without a schema change. The list of supported values is returned by `GET /api/sports`.
The frontend builds its dropdowns from that endpoint, not from hardcoded values.

Initial values:

| sport | movement | Display name |
|---|---|---|
| `volleyball` | `serve` | Serve |
| `volleyball` | `spike` | Spike |
| `volleyball` | `jump` | Jump |

### 3.4 Error severity

`info` < `warning` < `critical`. The frontend uses it for colors and sorting (critical first).

### 3.5 Movement phases [Draft]

Optional field `phase` in a technical error tells the user in which part of the movement the mistake
happened. Allowed values are defined per movement by the AI service. Examples: `approach`, `takeoff`,
`swing`, `contact`, `landing`. The frontend must treat unknown values as plain text.

## 4. Endpoints

Summary:

| Method | Path | Purpose | Marker |
|---|---|---|---|
| GET | `/health` | Liveness check | MVP |
| GET | `/api/sports` | Catalog of sports and movements | MVP |
| POST | `/api/analyses` | Upload a video, create an analysis | MVP |
| GET | `/api/analyses/{analysis_id}` | Status and metadata | MVP |
| GET | `/api/analyses/{analysis_id}/results` | Score, errors, recommendations | MVP |
| GET | `/api/analyses/{analysis_id}/landmarks` | Per-frame skeleton coordinates | MVP |
| GET | `/api/analyses/{analysis_id}/video` | Original video for the player | MVP |
| DELETE | `/api/analyses/{analysis_id}` | Delete an analysis and its files | MVP |
| GET | `/api/analyses` | List analyses (history page) | Planned |

---

### 4.1 `GET /health` [MVP]

Used by Docker and monitoring.

`200 OK`
```json
{ "status": "ok" }
```

---

### 4.2 `GET /api/sports` [MVP]

Returns the catalog used to build the upload form.

`200 OK`
```json
{
  "sports": [
    {
      "code": "volleyball",
      "name": "Volleyball",
      "movements": [
        { "code": "serve", "name": "Serve" },
        { "code": "spike", "name": "Spike" },
        { "code": "jump", "name": "Jump" }
      ]
    }
  ]
}
```

---

### 4.3 `POST /api/analyses` [MVP]

Uploads a video and creates an analysis.

**Request:** `multipart/form-data`

| Field | Type | Required | Description |
|---|---|---|---|
| `video` | file | yes | The video file (see limits in section 7) |
| `sport` | string | yes | Code from `/api/sports`, for example `volleyball` |
| `movement` | string | yes | Code from `/api/sports`, for example `spike` |

**Response:** `202 Accepted`

Headers: `Location: /api/analyses/3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77`

```json
{
  "analysis_id": "3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77",
  "status": "queued",
  "sport": "volleyball",
  "movement": "spike",
  "created_at": "2026-11-03T12:00:00Z",
  "links": {
    "status": "/api/analyses/3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77",
    "results": "/api/analyses/3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77/results",
    "landmarks": "/api/analyses/3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77/landmarks",
    "video": "/api/analyses/3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77/video"
  }
}
```

**Errors:** `400 validation_error`, `413 file_too_large`, `415 unsupported_media_type`,
`422 video_invalid`, `429 too_many_requests`.

Notes:
- The response is returned immediately after the file is stored. Processing happens in the background.
- The frontend should use the `links` values instead of building URLs by hand.
- Upload progress is measured on the client (see section 8.2).

---

### 4.4 `GET /api/analyses/{analysis_id}` [MVP]

Returns the current status. This is the endpoint the frontend polls.

`200 OK`
```json
{
  "analysis_id": "3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77",
  "status": "processing",
  "sport": "volleyball",
  "movement": "spike",
  "original_filename": "spike_training.mp4",
  "progress": 10,
  "stage": "analyzing",
  "created_at": "2026-11-03T12:00:00Z",
  "started_at": "2026-11-03T12:00:04Z",
  "completed_at": null,
  "expires_at": "2026-11-10T12:00:00Z",
  "error": null,
  "links": {
    "status": "/api/analyses/3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77",
    "results": "/api/analyses/3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77/results",
    "landmarks": "/api/analyses/3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77/landmarks",
    "video": "/api/analyses/3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77/video"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `status` | enum | `queued`, `processing`, `completed`, `failed` |
| `progress` | integer or null | 0 to 100. **Coarse in the MVP** (0 queued, 10 processing, 100 completed). The frontend must not assume smooth progress, show an indeterminate spinner while `processing` |
| `stage` | string or null | `queued`, `analyzing`, `saving`, `done`. For display only, may change |
| `started_at` / `completed_at` | timestamp or null | `null` until reached |
| `expires_at` | timestamp | After this moment the data is deleted (retention [Draft]: 7 days) |
| `error` | object or null | Filled only when `status` is `failed` |

`error` object when `failed`:
```json
{
  "code": "ai_video_unreadable",
  "message": "The video file could not be processed."
}
```

**Errors:** `404 not_found`, `410 expired`.

---

### 4.5 `GET /api/analyses/{analysis_id}/results` [MVP]

Returns the analysis outcome **without** per-frame coordinates (those are large and have their own
endpoint). This is what the results dashboard needs.

**Response:** `200 OK`, schema in section 5.1.

**Errors:** `404 not_found`, `409 result_not_ready` (status is `queued` or `processing`),
`409 analysis_failed` (status is `failed`), `410 expired`.

Caching: a completed result never changes. The response carries an `ETag`, the client may cache it.

---

### 4.6 `GET /api/analyses/{analysis_id}/landmarks` [MVP]

Returns skeleton coordinates for every frame. This is what the player overlay needs.

**Query parameters (optional)**

| Parameter | Type | Description |
|---|---|---|
| `from_frame` | integer | First frame index to return (inclusive). Default `0` |
| `to_frame` | integer | Last frame index to return (inclusive). Default last frame |

**Response:** `200 OK`, schema in section 5.2. The response is gzip-compressed.

**Errors:** `400 validation_error`, `404 not_found`, `409 result_not_ready`, `409 analysis_failed`,
`410 expired`.

Size note: roughly 150 frames of 33 points is about 350 KB uncompressed, a 60-second video at
30 fps is several MB. Because of this the frontend loads landmarks **once**, keeps them in memory
and does not re-request them while the video plays. A more compact encoding is [Planned] (section 5.2).

---

### 4.7 `GET /api/analyses/{analysis_id}/video` [MVP]

Streams the originally uploaded video for the `<video>` element.

- Content type is the real video type, for example `video/mp4`.
- **HTTP Range requests are supported** (`Accept-Ranges: bytes`, `206 Partial Content`),
  so seeking works in all browsers.
- Use it directly as the `src` of the player:
  `<video src={`${API_BASE}${analysis.links.video}`} />`

**Errors:** `404 not_found`, `410 expired`.

---

### 4.8 `DELETE /api/analyses/{analysis_id}` [MVP]

Deletes the analysis record, the video file and the results.

`204 No Content`

**Errors:** `404 not_found`. Deleting an analysis that is `processing` is allowed, the backend cancels
or discards the work.

---

### 4.9 `GET /api/analyses` [Planned]

Lists analyses for a history page. Without authentication this would show everyone's analyses, so it
is **not part of the MVP**. Until it exists, the frontend may keep the ids of the user's own analyses
in `localStorage`.

**Query parameters:** `page` (default 1), `page_size` (default 20, max 100),
`status`, `sport`, `movement`, `sort` (`created_at` or `-created_at`, default `-created_at`).

`200 OK`
```json
{
  "items": [ /* same objects as GET /api/analyses/{id} */ ],
  "page": 1,
  "page_size": 20,
  "total_items": 1,
  "total_pages": 1
}
```

## 5. Data schemas

### 5.1 Results (`GET .../results`)

```json
{
  "schema_version": "1.0",
  "analysis_id": "3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77",
  "status": "completed",
  "sport": "volleyball",
  "movement": "spike",
  "video": {
    "fps": 30,
    "duration_sec": 5.0,
    "frame_count": 150,
    "width": 1080,
    "height": 1920
  },
  "detection": {
    "frames_with_pose": 146,
    "detection_rate": 0.973
  },
  "overall_score": null,
  "score_breakdown": [],
  "metrics": [],
  "errors": [],
  "recommendations": [],
  "completed_at": "2026-11-03T12:00:41Z"
}
```

Until the scoring algorithm exists, `overall_score` is `null` and `errors`, `metrics`,
`score_breakdown`, `recommendations` are empty arrays. **The frontend must render these empty states**
(for example "Score is not available yet"). The backend never invents scores or errors.

| Field | Type | Description |
|---|---|---|
| `schema_version` | string | Version of this result format, currently `"1.0"` |
| `video.*` | numbers | Properties of the processed video. `width`/`height` are the **displayed** size (after rotation metadata is applied) |
| `detection.frames_with_pose` | integer | Number of frames where a person was found |
| `detection.detection_rate` | number | `frames_with_pose / frame_count`, 0 to 1. A low value means a poor video, the frontend may show a warning below 0.5 |
| `overall_score` | number or null | 0 to 100, or `null` when not available |
| `score_breakdown` | array | Optional details of the score, see below |
| `metrics` | array | Measured values (angles, speeds), see below |
| `errors` | array | Technical errors found, see below |
| `recommendations` | array | General advice, see below |

**`score_breakdown[]`**
```json
{ "criterion": "arm_swing", "label": "Arm swing", "score": 72.5, "weight": 0.3 }
```

**`metrics[]`**
```json
{
  "code": "right_elbow_angle_at_contact",
  "label": "Right elbow angle at contact",
  "value": 168.4,
  "unit": "deg",
  "frame_index": 87
}
```
`unit` is one of `deg`, `sec`, `m`, `m_s`, `ratio`. `frame_index` may be `null`.

**`errors[]`** (technical errors)
```json
{
  "code": "elbow_too_low",
  "title": "Elbow drops too low",
  "description": "The hitting elbow is below shoulder level during the swing.",
  "severity": "warning",
  "phase": "swing",
  "start_frame": 62,
  "end_frame": 78,
  "start_time_sec": 2.07,
  "end_time_sec": 2.6,
  "joints": ["right_elbow", "right_shoulder"],
  "recommendation": "Keep the elbow high and in line with the shoulder."
}
```
- `code` is **stable** and is the key for localization. `title`, `description` and `recommendation`
  are English default texts, the frontend may replace them by its own translations keyed by `code`.
- `phase`, `start_frame`, `end_frame`, `start_time_sec`, `end_time_sec`, `joints`, `recommendation`
  may be `null` (or an empty array for `joints`).
- Clicking an error in the dashboard should seek the player to `start_time_sec` and highlight `joints`.

**`recommendations[]`** (general advice, not tied to one error)
```json
{
  "code": "improve_approach_speed",
  "text": "Work on a faster last two steps of the approach.",
  "priority": 1
}
```
`priority`: 1 is the most important.

> The example values above only demonstrate the structure. Real values appear when the AI algorithm
> is implemented.

---

### 5.2 Landmarks (`GET .../landmarks`)

```json
{
  "schema_version": "1.0",
  "analysis_id": "3f2a9c1e-7b4d-4c3a-9a51-2f6d8e0b1c77",
  "fps": 30,
  "frame_count": 150,
  "frames": [
    {
      "frame_index": 0,
      "timestamp_sec": 0.0,
      "pose_detected": true,
      "landmarks": {
        "nose": { "x": 0.52, "y": 0.18, "z": -0.31, "visibility": 0.99 },
        "right_elbow": { "x": 0.31, "y": 0.30, "z": -0.15, "visibility": 0.98 }
      },
      "angles": null
    },
    {
      "frame_index": 1,
      "timestamp_sec": 0.033,
      "pose_detected": false,
      "landmarks": null,
      "angles": null
    }
  ]
}
```

(The example shows two landmarks for brevity, a real frame contains all 33.)

**Rules**

- `frames` always contains **one entry per frame**, in order, with no gaps, so `frames[i].frame_index === i`
  when `from_frame` is `0`.
- If no person is found, `pose_detected` is `false` and `landmarks` is `null`. The overlay must hide
  the skeleton for that frame. It must not crash.
- `x` and `y` are **normalized to 0..1** relative to the displayed video frame:
  `x = 0` is the left edge, `x = 1` is the right edge, `y = 0` is the top, `y = 1` is the bottom.
  Values slightly outside 0..1 are possible when a body part leaves the frame.
- `z` is relative depth (smaller means closer to the camera), roughly on the same scale as `x`.
  It is not needed for 2D drawing.
- `visibility` is 0..1, the confidence that the point is visible. Recommended threshold for drawing: **0.5**.
- `angles` is `null` until joint angles are implemented, later an object such as
  `{ "right_elbow": 142.5, "left_knee": 171.0 }` in degrees.
- `timestamp_sec = frame_index / fps`.

**The 33 landmark names (index order)**

| # | name | # | name | # | name |
|---|---|---|---|---|---|
| 0 | `nose` | 11 | `left_shoulder` | 22 | `right_thumb` |
| 1 | `left_eye_inner` | 12 | `right_shoulder` | 23 | `left_hip` |
| 2 | `left_eye` | 13 | `left_elbow` | 24 | `right_hip` |
| 3 | `left_eye_outer` | 14 | `right_elbow` | 25 | `left_knee` |
| 4 | `right_eye_inner` | 15 | `left_wrist` | 26 | `right_knee` |
| 5 | `right_eye` | 16 | `right_wrist` | 27 | `left_ankle` |
| 6 | `right_eye_outer` | 17 | `left_pinky` | 28 | `right_ankle` |
| 7 | `left_ear` | 18 | `right_pinky` | 29 | `left_heel` |
| 8 | `right_ear` | 19 | `left_index` | 30 | `right_heel` |
| 9 | `mouth_left` | 20 | `right_index` | 31 | `left_foot_index` |
| 10 | `mouth_right` | 21 | `left_thumb` | 32 | `right_foot_index` |

`left` and `right` are the **person's** left and right, not the viewer's.

**Skeleton connections to draw (body only, pairs of names)**

```
left_shoulder  – right_shoulder
left_shoulder  – left_elbow      left_elbow  – left_wrist
right_shoulder – right_elbow     right_elbow – right_wrist
left_shoulder  – left_hip        right_shoulder – right_hip
left_hip       – right_hip
left_hip       – left_knee       left_knee   – left_ankle
right_hip      – right_knee      right_knee  – right_ankle
left_ankle     – left_heel       left_heel   – left_foot_index
left_ankle     – left_foot_index
right_ankle    – right_heel      right_heel  – right_foot_index
right_ankle    – right_foot_index
```
Face and finger connections are optional. The full official list is `POSE_CONNECTIONS` in MediaPipe.

**Compact format [Planned]**

For long videos the backend will support `?format=compact`: `landmark_names` (33 names) once at the top
and each frame's `landmarks` as an array of 33 `[x, y, z, visibility]` arrays in index order. The default
format stays the object format described above.

## 6. Errors

### 6.1 Error response format

Every error response (any non-2xx status) has the same body:

```json
{
  "error": {
    "code": "unsupported_media_type",
    "message": "Only .mp4 and .mov files are supported.",
    "details": [],
    "request_id": "0HN7K2R4F1A0B"
  }
}
```

- `code`: stable machine-readable string. The frontend branches on `code`, never on `message`.
- `message`: English text for developers and fallback display.
- `details`: array, filled for validation errors, otherwise empty.
- `request_id`: the same value as the `X-Request-Id` header.

`details` item for validation errors:
```json
{ "field": "movement", "code": "invalid_value", "message": "Unknown movement 'smash' for sport 'volleyball'." }
```

### 6.2 Error catalog

| HTTP | `code` | When |
|---|---|---|
| 400 | `validation_error` | Missing or invalid field (see `details`) |
| 400 | `invalid_frame_range` | `from_frame` / `to_frame` out of range |
| 404 | `not_found` | Unknown `analysis_id` |
| 409 | `result_not_ready` | Results requested while `queued` or `processing` |
| 409 | `analysis_failed` | Results requested for a `failed` analysis |
| 410 | `expired` | Analysis data was deleted after the retention period |
| 413 | `file_too_large` | File exceeds the size limit |
| 415 | `unsupported_media_type` | File type is not allowed |
| 422 | `video_invalid` | File is not a readable video, too long, or too short |
| 429 | `too_many_requests` | Too many uploads or active analyses, see `Retry-After` |
| 500 | `internal_error` | Unexpected backend error |
| 503 | `service_unavailable` | Backend or its dependency is temporarily down, see `Retry-After` |

### 6.3 Processing errors (`error.code` inside a `failed` analysis)

These are not HTTP errors, they appear in `GET /api/analyses/{id}` → `error` when `status = failed`.

| `error.code` | Meaning | Suggested user message |
|---|---|---|
| `ai_video_unreadable` | AI could not decode the video | "We couldn't read this video. Try another file." |
| `ai_no_person_detected` | No person found in the whole video | "We couldn't find a person in the video." |
| `ai_timeout` | Processing took too long | "Processing took too long. Try a shorter video." |
| `ai_unavailable` | AI service did not respond | "The analysis service is unavailable. Try again later." |
| `internal_error` | Unexpected failure | "Something went wrong. Try again." |

The frontend maps `error.code` to its own localized text and falls back to `error.message`.

## 7. Limits and validation [Draft]

| Rule | Value |
|---|---|
| Allowed formats | `.mp4`, `.mov` (MIME `video/mp4`, `video/quicktime`) |
| Max file size | 200 MB |
| Max duration | 60 seconds |
| Min duration | 1 second |
| People in the video | One athlete. If several are visible, the AI tracks the most prominent person |
| Retention | Analysis data is kept for 7 days, then removed (`expires_at`) |
| Concurrent active analyses | Limited globally, excess uploads get `429` with `Retry-After` |

The frontend should **validate type and size before uploading** (to save the user's time) but must
still handle the backend errors above, because the backend is the authority.

Video recorded on phones often carries rotation metadata. The AI service applies it, so the
coordinates in `landmarks` and `video.width`/`video.height` always match what the browser displays.

## 8. Frontend implementation guide

### 8.1 Screens and the API calls behind them

| Screen | Calls |
|---|---|
| Upload form | `GET /api/sports` (dropdowns), `POST /api/analyses` |
| Processing screen | `GET /api/analyses/{id}` repeatedly |
| Result screen | `GET .../results`, `GET .../landmarks`, `GET .../video` |
| Delete button | `DELETE /api/analyses/{id}` |

### 8.2 Upload with progress

`fetch` does not report upload progress. Use `XMLHttpRequest` or axios (`onUploadProgress`).

```ts
const form = new FormData();
form.append("video", file);
form.append("sport", "volleyball");
form.append("movement", "spike");

const response = await axios.post<CreateAnalysisResponse>(`${API_BASE}/api/analyses`, form, {
  onUploadProgress: (e) => setUploadPercent(Math.round((e.loaded * 100) / (e.total ?? file.size))),
});
```

Do not set `Content-Type` manually for `FormData`, the browser adds the boundary itself.
After `202`, navigate to the processing screen using `analysis_id`.

### 8.3 Polling the status

- Request `GET /api/analyses/{id}` every **2 seconds**; after 30 seconds slow down to every **5 seconds**.
- Stop when `status` is `completed` or `failed`.
- Stop after **10 minutes** and show "This is taking longer than expected".
- Pause polling when the tab is hidden (`document.visibilityState`) and resume when it is visible again.
- On a network error or `503`, retry with a short backoff (2s, 4s, 8s), give up after 5 attempts.
- On `404` or `410`, stop and show "Analysis not found or expired".
- With TanStack Query: `refetchInterval` returning `false` when the status is terminal.

### 8.4 Status to UI mapping

| Condition | UI |
|---|---|
| Uploading | Progress bar with percent |
| `queued` | "Waiting in queue" with spinner |
| `processing` | "Analysing your technique" with indeterminate spinner |
| `completed` | Open the result screen |
| `failed` | Error message from `error.code`, button "Upload another video" |
| `completed` but `overall_score` is `null` | Show errors and recommendations if present, otherwise "Detailed assessment is not available yet" |
| `detection_rate` below 0.5 | Warning "The athlete is hard to see in this video, results may be inaccurate" |

### 8.5 Drawing the skeleton over the video

1. Place a `<canvas>` exactly over the `<video>` (same size, `position: absolute`, `pointer-events: none`).
2. Keep the canvas size equal to the video's **displayed** size. Recompute on window resize.
   Important: if the video is letterboxed (`object-fit: contain`), compute the real video rectangle
   inside the element, otherwise the skeleton will be shifted.
3. Convert normalized coordinates to pixels: `px = x * drawnWidth + offsetX`, `py = y * drawnHeight + offsetY`.
4. Choose the frame for the current time:
   `frameIndex = Math.min(frames.length - 1, Math.round(video.currentTime * fps))`.
5. Redraw on every displayed video frame. Prefer `video.requestVideoFrameCallback()` and fall back
   to `requestAnimationFrame` where it is unavailable (use `metadata.mediaTime` for the time).
6. Skip points with `visibility < 0.5` and skip connections where either end is skipped.
7. If `pose_detected` is `false`, clear the canvas for that frame.
8. Load landmarks once and keep them in memory (an array indexed by `frame_index`). Do not request
   them per frame.

## 9. TypeScript types

```ts
export type AnalysisStatus = "queued" | "processing" | "completed" | "failed";
export type Severity = "info" | "warning" | "critical";

export interface Links {
  status: string;
  results: string;
  landmarks: string;
  video: string;
}

export interface ApiErrorBody {
  error: {
    code: string;
    message: string;
    details: { field: string; code: string; message: string }[];
    request_id: string;
  };
}

export interface Sport {
  code: string;
  name: string;
  movements: { code: string; name: string }[];
}
export interface SportsResponse { sports: Sport[] }

export interface CreateAnalysisResponse {
  analysis_id: string;
  status: AnalysisStatus;
  sport: string;
  movement: string;
  created_at: string;
  links: Links;
}

export interface Analysis {
  analysis_id: string;
  status: AnalysisStatus;
  sport: string;
  movement: string;
  original_filename: string;
  progress: number | null;
  stage: string | null;
  created_at: string;
  started_at: string | null;
  completed_at: string | null;
  expires_at: string;
  error: { code: string; message: string } | null;
  links: Links;
}

export interface VideoInfo {
  fps: number;
  duration_sec: number;
  frame_count: number;
  width: number;
  height: number;
}

export interface TechniqueError {
  code: string;
  title: string;
  description: string;
  severity: Severity;
  phase: string | null;
  start_frame: number | null;
  end_frame: number | null;
  start_time_sec: number | null;
  end_time_sec: number | null;
  joints: string[];
  recommendation: string | null;
}

export interface Metric {
  code: string;
  label: string;
  value: number;
  unit: "deg" | "sec" | "m" | "m_s" | "ratio";
  frame_index: number | null;
}

export interface ScoreItem {
  criterion: string;
  label: string;
  score: number;
  weight: number;
}

export interface Recommendation {
  code: string;
  text: string;
  priority: number;
}

export interface AnalysisResults {
  schema_version: string;
  analysis_id: string;
  status: "completed";
  sport: string;
  movement: string;
  video: VideoInfo;
  detection: { frames_with_pose: number; detection_rate: number };
  overall_score: number | null;
  score_breakdown: ScoreItem[];
  metrics: Metric[];
  errors: TechniqueError[];
  recommendations: Recommendation[];
  completed_at: string;
}

export interface Landmark { x: number; y: number; z: number; visibility: number }

export interface Frame {
  frame_index: number;
  timestamp_sec: number;
  pose_detected: boolean;
  landmarks: Record<string, Landmark> | null;
  angles: Record<string, number> | null;
}

export interface LandmarksResponse {
  schema_version: string;
  analysis_id: string;
  fps: number;
  frame_count: number;
  frames: Frame[];
}
```

## 10. Mocking and dev workflow

The frontend can start **before** the backend is ready:

1. Use **MSW (Mock Service Worker)** or `json-server` and mock the endpoints from section 4 with the
   examples in this document. Include mocks for the failure cases (`failed`, `409`, `413`, `pose_detected: false`).
2. When the backend skeleton is available, it will run in a **fake AI mode**
   (`Ai__UseFake=true`) that returns a prepared result for any uploaded video. This lets the frontend
   test the full flow (upload, polling, results) against the real API before the real AI exists.
3. Run everything together with `docker compose up` from `octarsport-general`.
4. The real API is always inspectable at `/swagger`. If Swagger and this document disagree,
   **open an issue**, one of them is a bug.

Suggested sample data to keep in the frontend repo: one completed result with `errors` filled,
one with empty arrays and `overall_score: null`, one landmarks file with a few `pose_detected: false` frames.

## 11. Versioning and change process

- `schema_version` (inside results and landmarks) changes when the **data format** changes incompatibly.
- Backward-compatible changes (new optional fields, new endpoints, new enum values for `sport`, `movement`,
  `phase`, `unit`) do **not** change the version. The client must ignore unknown fields and handle unknown
  string values gracefully.
- A breaking change to endpoints introduces `/api/v2` and keeps `/api` working until the frontend migrates.
- Process: open a Pull Request in `octarsport-general` that edits this file, tag both owners, merge only
  after both approve. If the change affects the AI service, Artur is a required reviewer too.
- The backend implements the contract. Swagger is generated from the code, this document describes the
  intent. They must match.

## 12. Open questions

Items to discuss and close between Adrian and Yura:

1. **History page.** Do we need a list of previous analyses in the MVP? If yes, we need identification
   of the user (even a simple anonymous id) first. For now, `localStorage` with saved `analysis_id`s.
2. **Separate `results` and `landmarks`.** Is this split acceptable for the frontend, or is one
   combined response easier?
3. **Limits.** Are 200 MB, 60 seconds and `.mp4`/`.mov` realistic for the target videos?
   Should `.webm` or `.avi` be added?
4. **Localization.** Does the frontend want to localize texts by `code` (recommended), and which
   languages are needed (Ukrainian, English)?
5. **Progress.** Is an indeterminate spinner enough while processing, or is real progress needed?
   Real progress requires extra work on the AI side.
6. **Result retention.** Is 7 days acceptable?
7. **Push instead of polling.** Is polling acceptable for the MVP, with SSE/SignalR later?
8. **Error texts.** Which fields of an error does the UI actually need (phase, joints, time range)?
   Unused ones can be dropped.

## 13. Readiness checklist

**Backend (Adrian)**
- [ ] ASP.NET Core project with Swagger and CORS for `http://localhost:5173`
- [ ] `GET /health`, `GET /api/sports`
- [ ] `POST /api/analyses` with validation, storage and `202` response
- [ ] Status endpoint and background worker with fake AI mode
- [ ] `results`, `landmarks`, `video` (with Range) and `DELETE` endpoints
- [ ] Unified error format with `X-Request-Id`
- [ ] Docker Compose with PostgreSQL

**Frontend (Yura)**
- [ ] Mocks for all endpoints (MSW or json-server)
- [ ] Upload form with client-side validation and progress
- [ ] Polling with terminal states, timeouts and error handling
- [ ] Results screen: errors, score, recommendations, empty states
- [ ] Player with skeleton overlay synchronized by timestamps
- [ ] Mobile layout
- [ ] Types from section 9 in the codebase
