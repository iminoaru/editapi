FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg curl ca-certificates build-essential \
    fonts-noto fonts-noto-cjk fonts-noto-color-emoji \
    && rm -rf /var/lib/apt/lists/*

RUN curl -LsSf https://astral.sh/uv/install.sh | sh
ENV PATH="/root/.local/bin:${PATH}"

WORKDIR /app

COPY pyproject.toml /app/pyproject.toml
COPY uv.lock /app/uv.lock

RUN uv pip install --system fastapi uvicorn[standard] python-multipart sqlalchemy alembic psycopg2-binary pydantic pydantic-settings httpx python-jose[cryptography] passlib[bcrypt]

COPY app /app/app
COPY alembic /app/alembic
COPY alembic.ini /app/alembic.ini
COPY docker /app/docker
COPY openapi-export.sh /app/openapi-export.sh


RUN mkdir -p /data
EXPOSE 8000
