# Backend Video Processing — Execution Plan (Paste into Cursor)

> **Mode**: All-local, Dockerized. No external accounts.
> **Stack**: FastAPI (Python ≥3.11, `uv`), Postgres (via SQLAlchemy + Alembic), ffmpeg/ffprobe, Background jobs (ThreadPoolExecutor).
> **Async**: Non-blocking endpoints return `job_id`; background workers handle video work; job state persisted in Postgres.
> **Typing**: Modern Python (`list[str] | None` etc.).
> **Storage**: Local disk under `/data` (volume). Fonts in `/fonts`. Read-only assets at `/app/assets`.

---

## 🔒 Hard Rules (non-negotiable)

1. **Local-only**: Docker containers for API + DB. No Redis/Celery.
2. **Async processing**: Heavy work runs in an in-process job runner (ThreadPoolExecutor). Endpoints immediately return `job_id`.
3. **DB is source of truth** for jobs (`PENDING | STARTED | SUCCESS | FAILURE`, progress, errors, I/O variant IDs).
4. **No overwrite**: Write outputs to `*.tmp` then atomic rename.
5. **Deterministic storage** under `/data/{uploads,processed,variants}`; fonts under `/fonts`; assets in `/app/assets` (read-only).
6. **Clean UX**: Helpful 4xx errors; 500 only for unknowns.
7. **FFmpeg transparency**: Log every command and stderr snippet on failure.
8. **OpenAPI polished**: Tags, summaries, examples.
9. **One command up**: `docker compose up --build` → `http://localhost:8000/docs`.
10. **Security**: No path traversal; only allow overlay assets within `/app/assets` or previously stored files in `/data`.

---

## 📁 Repository Layout

```
backend/
  app/
    main.py
    core/
      config.py            # env, paths, settings
      logging.py           # structured logger
      errors.py            # exception helpers
    db/
      base.py              # engine/session factory
      models.py            # ORM models + enums
      schemas.py           # Pydantic I/O schemas
      crud.py              # typed DB helpers
    api/
      deps.py              # DB deps, pagination
      routes_videos.py     # upload, list
      routes_trim.py       # POST /trim
      routes_overlays.py   # POST /overlays (text/image/video), watermark
      routes_jobs.py       # GET /status/{job_id}, GET /result/{job_id}
      routes_variants.py   # POST /transcode, list & download variants
    services/
      storage.py           # FS saving/streaming utils
      ffmpeg.py            # probe/trim/overlay/transcode
      filters.py           # build filter_complex for overlays
      jobs.py              # ThreadPool JobManager + handlers
    utils/
      ids.py               # UUIDs, safe filenames
      timecodes.py         # parse & clamp HH:MM:SS(.sss) / float
    lifecycle/
      startup.py           # init/shutdown hooks (executor lifecycle)
  alembic/
    env.py
    versions/
  assets/                  # local logos/overlays (read-only in container)
  docker/
    api-entrypoint.sh
  data/                    # (gitignored) mounted volume target
  fonts/                   # (gitignored) mounted fonts (Noto/Indic)
  .env.example
  docker-compose.yml
  Dockerfile
  pyproject.toml
  README.md
  openapi-export.sh
```

---

## 🔧 Environment (`.env.example`)

```
APP_ENV=dev
APP_HOST=0.0.0.0
APP_PORT=8000

DB_HOST=db
DB_PORT=5432
DB_NAME=videos
DB_USER=videos
DB_PASSWORD=videos

MEDIA_ROOT=/data
FONT_DIR=/fonts

FFMPEG_BIN=ffmpeg
FFPROBE_BIN=ffprobe

API_ROOT_PATH=
```

Copy to `.env` (unchanged for local).

---

## 🐳 Docker

**docker-compose.yml**

```yaml
version: "3.9"

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: videos
      POSTGRES_USER: videos
      POSTGRES_PASSWORD: videos
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U videos -d videos"]
      interval: 5s
      timeout: 5s
      retries: 20

  api:
    build:
      context: .
      dockerfile: Dockerfile
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    ports: ["8000:8000"]
    volumes:
      - ./data:/data
      - ./fonts:/fonts
      - ./assets:/app/assets:ro
    command: ["/bin/bash", "/app/docker/api-entrypoint.sh"]

volumes:
  pgdata:
```

**Dockerfile**

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg curl ca-certificates build-essential \
    fonts-noto fonts-noto-cjk fonts-noto-color-emoji \
    && rm -rf /var/lib/apt/lists/*

RUN curl -LsSf https://astral.sh/uv/install.sh | sh
ENV PATH="/root/.local/bin:${PATH}"

WORKDIR /app

COPY pyproject.toml /app/pyproject.toml
RUN uv sync --frozen

COPY app /app/app
COPY alembic /app/alembic
COPY docker /app/docker
COPY README.md /app/README.md
COPY openapi-export.sh /app/openapi-export.sh

RUN mkdir -p /data
EXPOSE 8000
```

**docker/api-entrypoint.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail
sleep 2
uv run alembic upgrade head
exec uv run uvicorn app.main:app --host ${APP_HOST:-0.0.0.0} --port ${APP_PORT:-8000} --proxy-headers
```

(Ensure `chmod +x docker/api-entrypoint.sh`.)

---

## 📦 pyproject (deps)

```toml
[project]
name = "video-backend"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "fastapi",
  "uvicorn[standard]",
  "python-multipart",
  "sqlalchemy>=2.0",
  "alembic",
  "psycopg[binary]",
  "pydantic-settings",
  "httpx",
]

[tool.uv]
dev-dependencies = ["pytest", "ruff", "mypy", "anyio"]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.mypy]
python_version = "3.11"
strict = true
ignore_missing_imports = true
```

---

## 🗃️ Database Schema (via Alembic)

### Enums

* `variant_kind`: `'trim' | 'overlay' | 'watermark' | 'transcode'`
* `variant_quality`: `'source' | '1080p' | '720p' | '480p'`
* `job_type`: `'upload_probe' | 'trim' | 'overlay' | 'watermark' | 'transcode_multi'`
* `job_status`: `'PENDING' | 'STARTED' | 'SUCCESS' | 'FAILURE'`
* `overlay_type`: `'text' | 'image' | 'video' | 'watermark'`

### Tables

**videos**

* `id` UUID PK
* `original_filename` text
* `stored_path` text
* `size_bytes` bigint
* `duration_sec` numeric(10,3) nullable
* `mime_type` text
* `upload_time` timestamptz default now()

**video\_variants**

* `id` UUID PK
* `video_id` UUID FK → videos(id) ON DELETE CASCADE
* `kind` variant\_kind
* `quality` variant\_quality nullable
* `source_variant_id` UUID FK → video\_variants(id) nullable
* `stored_path` text
* `size_bytes` bigint
* `duration_sec` numeric(10,3)
* `created_at` timestamptz default now()
* `config_json` jsonb

**overlays**

* `id` UUID PK
* `video_id` UUID FK → videos(id)
* `variant_id` UUID FK → video\_variants(id) nullable
* `type` overlay\_type
* `payload_json` jsonb
* `created_at` timestamptz default now()

**jobs**

* `id` UUID PK
* `video_id` UUID nullable
* `input_variant_id` UUID nullable
* `output_variant_id` UUID nullable
* `type` job\_type
* `status` job\_status
* `progress` int default 0
* `error_message` text nullable
* `created_at` timestamptz default now()
* `updated_at` timestamptz default now()

### Indexes

* `video_variants(video_id)`
* `jobs(status, created_at)`

---

## 🧵 Background Job System

**Goal**: Submit work, return `job_id`, track progress/state in DB.

**`services/jobs.py`**

* `JobManager` (singleton):

  * Holds a `ThreadPoolExecutor(max_workers=2)`.
  * API: `submit(job_type: str, payload: dict, handler: Callable) -> UUID`

    * Creates DB job row with `PENDING`.
    * Wraps handler with progress updates:

      * Set `STARTED`, `progress=0`.
      * Provide `JobContext` with:

        * `report_progress(int)`,
        * `set_output_variant(UUID)`,
        * `fail(msg: str)`.
      * On success: `SUCCESS`, `progress=100`.
      * On failure: `FAILURE`, store `error_message`.
* Optional dictionary `{job_id: Future}` for introspection.
* **On app startup**: instantiate JobManager.
* **On shutdown**: cleanly `executor.shutdown(wait=False)`.

**Handlers (inside `services/jobs.py` or `services/handlers.py`)**

* `handle_trim(ctx, payload)`: trim, create variant, set output id.
* `handle_overlay(ctx, payload)`: build filtergraph, overlay, create variant + overlays rows.
* `handle_transcode_multi(ctx, payload)`: produce 1080/720/480 variants; update progress in chunks.
* (Optional) `handle_probe_on_upload` if you want to probe async; otherwise probe sync on upload.

---

## 🛠️ Services

**`services/storage.py`**

* `save_upload(file: UploadFile) -> StoredFile {path, size_bytes, mime_type}`
* `make_temp_and_final(category, ext) -> (tmp_path, final_path)`
* `commit_temp(tmp, final) -> None` (atomic rename)
* `open_stream(path) -> StreamingResponse`
* `exists(path) -> bool`

**`services/ffmpeg.py`**

* `probe(path) -> {duration_sec: float, size_bytes: int}`

  ```
  ffprobe -v error -select_streams v:0 -show_entries format=duration,size -of json INPUT
  ```
* `trim(input, start, end, output)`: (accurate re-encode)

  ```
  ffmpeg -y -i INPUT -ss {start} -to {end} \
    -c:v libx264 -preset veryfast -crf 18 \
    -c:a aac -b:a 128k OUTPUT
  ```
* `overlay(input, overlays, watermark, output)`:

  * Build `filter_complex` with `drawtext` (Indic `text_shaping=1`) and `overlay` chains.
  * Example watermark:

    ```
    ffmpeg -y -i INPUT -i WM.png -filter_complex \
    "[1]format=rgba,colorchannelmixer=aa=0.6[wm];[0][wm]overlay=W-w-20:H-h-20:enable='between(t,0,1e9)'" \
      -c:v libx264 -crf 20 -preset veryfast -c:a copy OUTPUT
    ```
* `transcode_multi(input, heights=[1080,720,480]) -> dict[int, path]`

  ```
  ffmpeg -y -i INPUT -vf "scale=-2:{height}" \
    -c:v libx264 -crf {crf} -preset veryfast \
    -c:a aac -b:a 128k OUTPUT_{height}p.mp4
  ```

  CRFs: 1080→20, 720→22, 480→24.

**`services/filters.py`** (overlay filtergraph)

* Accept Overlay specs: Text/Image/Video with x/y, start/end, font/size/color/opacity/scale.
* Build `-filter_complex` and extra `-i` inputs as needed.
* Map audio `-map 0:a?` and `-shortest`.

---

## 🧠 API Surface

### Level 1 — Upload & Metadata

**POST `/videos/upload`** (multipart)

* in: `file: UploadFile`
* do: save → ffprobe **inline** → insert into `videos`
* out: `VideoOut {id, original_filename, stored_path, size_bytes, duration_sec, mime_type, upload_time}`

**GET `/videos?page=&page_size=`**

* out: Paginated list of `VideoOut` ordered by `upload_time DESC`.

### Level 2 — Trimming

**POST `/trim`**

* Body:

```json
{
  "video_id": "UUID",
  "start": "00:00:00.500",
  "end": 2.0,
  "source_variant_id": null
}
```

* Validate timestamps against duration.
* Submit job: produce new variant `{kind: "trim", quality: "source"}`; link to original; save `config_json`.
* Out: `{ "job_id": "UUID" }`.

### Level 3 — Overlays & Watermark

**POST `/overlays`**

* Body (example):

```json
{
  "video_id": "UUID",
  "source_variant_id": null,
  "overlays": [
    {
      "type": "text",
      "text": "नमस्ते दुनिया",
      "font": "NotoSansDevanagari-Regular.ttf",
      "font_size": 48,
      "color": "white@0.85",
      "x": "w-tw-20",
      "y": "h-th-20",
      "start": 1.5,
      "end": 5.0
    },
    {
      "type": "image",
      "image_path": "/app/assets/logo.png",
      "x": 20,
      "y": 20,
      "start": 0,
      "end": 600,
      "opacity": 0.6
    }
  ],
  "watermark": {
    "image_path": "/app/assets/wm.png",
    "x": "W-w-20",
    "y": "H-h-20",
    "opacity": 0.5
  }
}
```

* Validate asset paths (must be in `/app/assets` or under `/data`).
* Submit job: create overlay/watermark variant; persist overlay rows with payload.

### Level 4 — Jobs

**GET `/status/{job_id}`**

* Out: `{ job_id, status, progress, error_message? }` from DB.

**GET `/result/{job_id}`**

* If SUCCESS → stream result file via `output_variant_id`.
* If STARTED/PENDING → 409.
* If FAILURE → 404 (reason from `/status`).

### Level 5 — Multiple Qualities

**POST `/transcode/{video_or_variant_id}`**

* Body:

```json
{ "qualities": ["1080p","720p","480p"] }
```

* Submit job: produce variants for each quality; attach to `video_id`.
* Out: `{ job_id }`.

**GET `/videos/{video_id}/variants`**

* Out: list of variants with `{id, kind, quality, stored_path, size_bytes, duration_sec, created_at}`.

**GET `/variants/{variant_id}/download`**

* Stream file.

---

## 🧪 Validation & Edge Cases

* Timestamps: allow float or `HH:MM:SS(.sss)`. Reject `end <= start`. Clamp within duration.
* Assets: deny paths outside `/app/assets` or `/data`. Prevent traversal (`..` etc.).
* Concurrency: `max_workers=2` to prevent overload.
* Restart semantics: (optional) On startup, mark lingering `STARTED` jobs as `FAILURE: server restarted`.

---

## 🧰 OpenAPI polish

* Add `tags` and `summary` to every route.
* Include example bodies as above.
* Script to export OpenAPI:

  ```bash
  ./openapi-export.sh
  ```

**openapi-export.sh**

```bash
#!/usr/bin/env bash
curl -sS http://localhost:8000/openapi.json -o openapi.json
echo "openapi.json exported."
```

---

## 🎬 Demo Recording Script

1. `docker compose up --build`
2. `http://localhost:8000/docs`
3. Upload a small mp4 via **POST /videos/upload** → verify metadata.
4. **POST /trim** (0.5s → 2.0s) → get `job_id`.
5. Poll **GET /status/{job\_id}** → `SUCCESS`.
6. **GET /result/{job\_id}** → save & play.
7. **POST /overlays** (Hindi text + watermark from `assets/`) → `job_id` → status → result → play.
8. **POST /transcode/{video\_id}** with `["480p","720p","1080p"]` → status → **GET /videos/{id}/variants** → **GET /variants/{variant\_id}/download** (480p) → play.
9. `./openapi-export.sh` → show `openapi.json`.

---

## ✅ Implementation Order

1. **Scaffold** directories/files as per layout.
2. **pyproject.toml** with deps; `uv sync`.
3. **core/config.py**: read `.env`, resolve paths.
4. **db/models.py**: enums + tables exactly as above.
5. **alembic**: init; write initial migration; `alembic upgrade head`.
6. **db/crud.py**: helpers for videos, variants, jobs, overlays (create/list/get/update).
7. **services/storage.py**: safe temp→rename writers; streaming.
8. **utils/timecodes.py**: parse/clamp helpers.
9. **services/ffmpeg.py**: probe/trim/overlay/transcode with subprocess + error handling.
10. **services/filters.py**: build overlay filter\_complex; manage additional `-i` inputs; map audio; `-shortest`.
11. **services/jobs.py**: JobManager (executor), JobContext, handlers (trim/overlay/transcode). Persist progress + outcomes in `jobs`.
12. **lifecycle/startup.py** + **app/main.py**: create JobManager on startup; graceful shutdown.
13. **api routes**:

    * `/videos/upload`, `/videos`
    * `/trim`
    * `/overlays`
    * `/transcode/{id}`, `/videos/{id}/variants`, `/variants/{id}/download`
    * `/status/{job_id}`, `/result/{job_id}`
      Add tags/summaries/examples.
14. **Docker**: Dockerfile, compose, entrypoint; bring up stack.
15. **README.md**: quickstart, demo steps, troubleshooting (fonts/ffmpeg).
16. **(Optional tests)**: generate a 3s sample via ffmpeg, run smoke tests.

---

## 🧾 Pydantic Schemas (reference)

```python
from pydantic import BaseModel, Field
from typing import Literal
from uuid import UUID

class VideoOut(BaseModel):
    id: UUID
    original_filename: str
    stored_path: str
    size_bytes: int
    duration_sec: float | None
    mime_type: str
    upload_time: str

class TrimIn(BaseModel):
    video_id: UUID
    start: float | str
    end: float | str
    source_variant_id: UUID | None = None

class OverlayText(BaseModel):
    type: Literal["text"] = "text"
    text: str
    font: str | None = None
    font_size: int | None = 32
    color: str | None = "white"
    x: int | str = 20
    y: int | str = 20
    start: float = 0
    end: float | None = None

class OverlayImage(BaseModel):
    type: Literal["image"] = "image"
    image_path: str
    x: int | str = 20
    y: int | str = 20
    start: float = 0
    end: float | None = None
    opacity: float | None = None

class OverlayVideo(BaseModel):
    type: Literal["video"] = "video"
    video_path: str
    x: int | str = 20
    y: int | str = 20
    start: float = 0
    end: float | None = None
    scale: float | None = 1.0

OverlaySpec = OverlayText | OverlayImage | OverlayVideo

class Watermark(BaseModel):
    image_path: str
    x: int | str = "W-w-20"
    y: int | str = "H-h-20"
    opacity: float = 0.5

class OverlaysIn(BaseModel):
    video_id: UUID
    source_variant_id: UUID | None = None
    overlays: list[OverlaySpec] = Field(default_factory=list)
    watermark: Watermark | None = None

class TranscodeIn(BaseModel):
    qualities: list[Literal["1080p","720p","480p"]] = ["1080p","720p","480p"]

class JobIdOut(BaseModel):
    job_id: UUID

class JobStatusOut(BaseModel):
    job_id: UUID
    status: Literal["PENDING","STARTED","SUCCESS","FAILURE"]
    progress: int
    error_message: str | None = None
```

---

## 🩹 Troubleshooting (README section)
* **Large files**: Document reverse-proxy limits if any (not needed for local demo).
* **Jobs stuck**: Check `docker logs` for ffmpeg stderr; `status` stays `STARTED` until finish/fail.
* **Path issues**: Overlay assets must be inside `/app/assets` (mounted from `./assets`) or under `/data`; reject anything else.

---
