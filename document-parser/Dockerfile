# syntax=docker/dockerfile:1
# =============================================================================
# Docling Studio — backend image (multi-target: remote / local)
#
# Usage:
#   docker build --target remote -t docling-studio-backend:remote .
#   docker build --target local  -t docling-studio-backend:local  .
# =============================================================================

# --- Base: common deps for both targets ---
FROM python:3.12-slim AS base

RUN apt-get update && apt-get install -y --no-install-recommends \
    poppler-utils \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
# Use CPU-only PyTorch wheels to avoid pulling ~1GB of NVIDIA/CUDA packages.
ENV PIP_EXTRA_INDEX_URL=https://download.pytorch.org/whl/cpu
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd --create-home --shell /bin/bash appuser \
    && mkdir -p /app/uploads /app/data \
    && chown -R appuser:appuser /app

EXPOSE 8000

ENV UPLOAD_DIR=/app/uploads
ENV DB_PATH=/app/data/docling_studio.db

USER appuser
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

# --- Remote: lightweight, delegates to Docling Serve ---
FROM base AS remote
ENV CONVERSION_ENGINE=remote

# --- Local: full Docling in-process ---
FROM base AS local

USER root
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgl1 \
    libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

COPY requirements-local.txt .
# Prefer CPU wheel index for PyTorch and avoid installing transitive CUDA/NVIDIA
# packages by installing PyTorch and docling without their deps. Base stage
# already installed common requirements from requirements.txt.
ENV PIP_EXTRA_INDEX_URL=https://download.pytorch.org/whl/cpu
RUN pip install --no-cache-dir --no-deps torch torchvision \
    && pip install --no-cache-dir --no-deps "docling>=2.80.0,<3.0.0"

RUN chown -R appuser:appuser /app \
    && chown -R appuser:appuser /usr/local/lib/python3.12/site-packages/rapidocr/models
USER appuser
ENV CONVERSION_ENGINE=local
