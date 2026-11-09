# Builds the NetLicensing MCP server the way its CI does: Python >= 3.12, installed from uv.lock with the dev extras
# (pytest, ruff, mypy, pip-audit). All wheels are fetched here, while the network is up, so the offline audit can run
# the server and the tests.
FROM python:3.12-slim-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PIP_ROOT_USER_ACTION=ignore \
    PIP_NO_CACHE_DIR=1
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git curl build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:0.8 /uv /usr/local/bin/uv

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# The version comes from the git tag (hatch-vcs). Pin one so the install does not depend on tags being present.
ENV SETUPTOOLS_SCM_PRETEND_VERSION_FOR_NETLICENSING_MCP=0.0.0
ENV UV_PROJECT_ENVIRONMENT=/src/.venv UV_LINK_MODE=copy PATH=/src/.venv/bin:$PATH

# Locked runtime + dev dependencies; the project itself is installed in editable mode.
RUN uv sync --frozen --extra dev

# Run the server's cheap checks so a broken image is visible in the build log without failing the build.
RUN (pytest -q --no-cov || echo "WARNING: pytest reported failures")
RUN (ruff check . || echo "WARNING: ruff reported findings")

# The server runs over stdio by default, or HTTP with `python -m netlicensing_mcp.server http` (127.0.0.1:8000).
# NETLICENSING_API_KEY is required unless NETLICENSING_ALLOW_DEMO=true; the sandbox has no network, so tests
# mock the NetLicensing API.
