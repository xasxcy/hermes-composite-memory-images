ARG PYTHON_BASE_IMAGE
FROM ${PYTHON_BASE_IMAGE}

ARG UV_IMAGE
ARG HONCHO_REPOSITORY=https://github.com/plastic-labs/honcho.git
ARG HONCHO_COMMIT=5ad22840d829878f9ac4d13e9538e5fef216c97e

COPY --from=${UV_IMAGE} /uv /bin/uv

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UV_LINK_MODE=copy \
    PATH="/app/.venv/bin:${PATH}"

RUN apt-get update \
    && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
RUN git clone "${HONCHO_REPOSITORY}" honcho \
    && cd honcho \
    && git checkout --detach "${HONCHO_COMMIT}" \
    && test "$(git rev-parse HEAD)" = "${HONCHO_COMMIT}"

WORKDIR /src/honcho
RUN uv sync --frozen --no-group dev

RUN addgroup --system app \
    && adduser --system --group app \
    && chown -R app:app /src/honcho

USER app
WORKDIR /src/honcho

LABEL org.opencontainers.image.source="https://github.com/plastic-labs/honcho" \
      org.opencontainers.image.revision="${HONCHO_COMMIT}"

EXPOSE 8000

ENTRYPOINT ["sh", "docker/entrypoint.sh"]
