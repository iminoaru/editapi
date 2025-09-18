# Video Processing Backend

A FastAPI-based backend for video processing with trimming, overlays, and transcoding capabilities.

## Features

- **Video Upload**: Upload video files with automatic metadata extraction
- **Video Trimming**: Trim videos to specified time ranges
- **Overlays**: Add text, image, and video overlays with precise positioning
- **Watermarks**: Apply semi-transparent watermarks
- **Transcoding**: Generate multiple quality variants (1080p, 720p, 480p)
- **Background Processing**: Async job processing with progress tracking
- **RESTful API**: Complete OpenAPI documentation

## Quick Start

### Prerequisites

- Docker and Docker Compose
- `uv` package manager (for local development)

### Running with Docker

1. **Clone and setup**:
   ```bash
   cd backend
   cp env.example .env
   ```

2. **Start the services**:
   ```bash
   docker compose up --build
   ```

3. **Access the API**:
   - API Documentation: http://localhost:8000/docs
   - Health Check: http://localhost:8000/

### Local Development

1. **Install dependencies**:
   ```bash
   uv sync
   ```

2. **Start PostgreSQL** (using Docker):
   ```bash
   docker run -d --name postgres \
     -e POSTGRES_DB=videos \
     -e POSTGRES_USER=videos \
     -e POSTGRES_PASSWORD=videos \
     -p 5432:5432 \
     postgres:16
   ```

3. **Run migrations**:
   ```bash
   uv run alembic upgrade head
   ```

4. **Start the API**:
   ```bash
   uv run uvicorn app.main:app --reload
   ```

## API Usage

### Upload Video
```bash
curl -X POST "http://localhost:8000/videos/upload" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@video.mp4"
```

### Trim Video
```bash
curl -X POST "http://localhost:8000/trim" \
  -H "Content-Type: application/json" \
  -d '{
    "video_id": "uuid-here",
    "start": "00:00:05",
    "end": "00:00:15"
  }'
```

### Apply Overlays
```bash
curl -X POST "http://localhost:8000/overlays" \
  -H "Content-Type: application/json" \
  -d '{
    "video_id": "uuid-here",
    "overlays": [
      {
        "type": "text",
        "text": "Hello World",
        "x": 20,
        "y": 20,
        "start": 0,
        "end": 5
      }
    ],
    "watermark": {
      "image_path": "/app/assets/logo.png",
      "opacity": 0.5
    }
  }'
```

### Check Job Status
```bash
curl "http://localhost:8000/jobs/status/{job_id}"
```

### Download Result
```bash
curl "http://localhost:8000/jobs/result/{job_id}" -o result.mp4
```

## Project Structure

```
backend/
├── app/
│   ├── api/                 # API routes
│   ├── core/               # Configuration and logging
│   ├── db/                 # Database models and CRUD
│   ├── services/           # Business logic services
│   ├── utils/              # Utility functions
│   └── lifecycle/         # App startup/shutdown
├── alembic/               # Database migrations
├── docker/               # Docker configuration
├── data/                 # Media storage (mounted volume)
├── fonts/                # Font files (mounted volume)
├── assets/               # Static assets (mounted volume)
└── docker-compose.yml    # Docker services
```

## Configuration

Environment variables (see `env.example`):

- `APP_ENV`: Environment (dev/prod)
- `APP_HOST`: API host (default: 0.0.0.0)
- `APP_PORT`: API port (default: 8000)
- `DB_*`: Database connection settings
- `MEDIA_ROOT`: Media storage directory
- `FONT_DIR`: Font directory
- `FFMPEG_BIN`: FFmpeg binary path
- `FFPROBE_BIN`: FFprobe binary path

## Troubleshooting

### Large Files
- The API handles large files through streaming
- Check Docker logs for FFmpeg errors: `docker logs video-editor-api-1`

### Jobs Stuck
- Check job status: `GET /jobs/status/{job_id}`
- Review logs for FFmpeg stderr output
- Jobs marked as `STARTED` will complete or fail

### Path Issues
- Overlay assets must be in `/app/assets` or `/data`
- Font files should be in `/fonts` directory
- All paths are validated for security

### Font Support
- Noto fonts are included for international text
- Custom fonts can be added to `/fonts` directory
- Text overlays support Indic text shaping

## Development

### Adding New Features
1. Create database models in `app/db/models.py`
2. Add CRUD operations in `app/db/crud.py`
3. Create API schemas in `app/db/schemas.py`
4. Implement business logic in `app/services/`
5. Add API routes in `app/api/`

### Database Migrations
```bash
# Create new migration
uv run alembic revision --autogenerate -m "Description"

# Apply migrations
uv run alembic upgrade head
```

### Testing
```bash
# Run tests
uv run pytest

# Export OpenAPI spec
./openapi-export.sh
```

## License

MIT License
