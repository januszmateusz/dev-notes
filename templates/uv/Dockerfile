# syntax=docker/dockerfile:1
ARG PYTHON_IMAGE=python:3.11-slim

FROM $PYTHON_IMAGE AS builder
COPY --from=ghcr.io/astral-sh/uv:0.12.19 /uv /uvx /bin/

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=0

WORKDIR /app

# warstwa 1: tylko zależności (zmienia się rzadko)
COPY pyproject.toml uv.lock .python-version ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-install-project

# warstwa 2: kod (zmienia się często)
COPY <package_name>/ ./<package_name>/
COPY data/ ./data/
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev


FROM $PYTHON_IMAGE AS runtime
RUN useradd --create-home --uid 1000 app
WORKDIR /app
COPY --from=builder --chown=app:app /app /app
ENV PATH="/app/.venv/bin:$PATH"
USER app
CMD ["python", "-m", "<package_name>.main"]