# syntax=docker/dockerfile:1
# What this does: it builds in a "workshop" image that has tools, then copies only the finished app into a bare "showroom" image. The Chainguard login is used only during the install and is never saved. The app runs as a non-root user with a proper production server.

# ---------- Build stage ----------
# Chainguard's -dev variant includes pip and a shell, needed only to install packages.
FROM cgr.dev/chainguard/python:latest-dev AS build

# Package index. Defaults to Chainguard Libraries (Chainguard-built packages only, no PyPI fallback).
# Can be pointed at a repository manager that proxies Chainguard Libraries.
ARG PIP_INDEX_URL=https://libraries.cgr.dev/python/simple/

# Root only in this throwaway stage so it can read build secrets. Only the venv is copied out.
USER root
WORKDIR /app
RUN python -m venv /app/venv
ENV PATH="/app/venv/bin:$PATH"

COPY requirements.txt .

# Credentials arrive as BuildKit secrets and are written to a temporary in-memory netrc,
# so they never land in an image layer or the build history.
RUN --mount=type=secret,id=cg_identity_id \
    --mount=type=secret,id=cg_token \
    --mount=type=tmpfs,target=/tmp/auth \
    set -e; \
    if [ -s /run/secrets/cg_identity_id ] && [ -s /run/secrets/cg_token ]; then \
      echo "Chainguard Libraries credentials: provided via build secrets"; \
      printf 'machine libraries.cgr.dev\nlogin %s\npassword %s\n' \
        "$(cat /run/secrets/cg_identity_id)" "$(cat /run/secrets/cg_token)" > /tmp/auth/netrc; \
    else \
      echo "Chainguard Libraries credentials: NOT provided"; \
    fi; \
    NETRC=/tmp/auth/netrc pip install --no-input --no-cache-dir --index-url "$PIP_INDEX_URL" -r requirements.txt

    # pip is only needed to install packages, not to run the app; remove it from the venv we ship.
RUN python -m pip uninstall -y pip

# ---------- Runtime stage ----------
# Minimal Chainguard image: no shell, no package manager.
FROM cgr.dev/chainguard/python:latest

WORKDIR /app
ENV PATH="/app/venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Files stay root-owned, so the app can read but not modify its own code or packages.
COPY --from=build /app/venv /app/venv
COPY app.py ./
COPY templates ./templates
COPY static ./static

# Chainguard's non-root user
USER 65532
EXPOSE 8000

# gunicorn (production server) instead of Flask's debug server.
# One worker with threads avoids a startup race on table creation; control socket disabled.
ENTRYPOINT ["python", "-m", "gunicorn", "--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", "--no-control-socket", "--access-logfile", "-", "app:app"]