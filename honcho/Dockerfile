ARG PYTHON_BASE_IMAGE
ARG UV_IMAGE
FROM ${UV_IMAGE} AS uv
FROM ${PYTHON_BASE_IMAGE}

ARG HONCHO_REPOSITORY=https://github.com/plastic-labs/honcho.git
ARG HONCHO_COMMIT=06ed1929cf017c333a87c41d130bb6a0605d91c0

COPY --from=uv /uv /bin/uv

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UV_LINK_MODE=copy \
    UV_PROJECT_ENVIRONMENT=/app/.venv \
    PATH="/app/.venv/bin:${PATH}"

RUN apt-get update \
    && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
COPY patches/ /tmp/honcho-patches/
RUN git clone "${HONCHO_REPOSITORY}" honcho \
    && cd honcho \
    && git checkout --detach "${HONCHO_COMMIT}" \
    && test "$(git rev-parse HEAD)" = "${HONCHO_COMMIT}" \
    && git apply --check /tmp/honcho-patches/0001-jwt-expiry-numericdate.patch \
    && git apply /tmp/honcho-patches/0001-jwt-expiry-numericdate.patch

WORKDIR /src/honcho
RUN uv sync --frozen --no-group dev --python /usr/local/bin/python \
    && test -x /app/.venv/bin/python \
    && /app/.venv/bin/python -c 'import src' \
    && AUTH_JWT_SECRET=01234567890123456789012345678901 /app/.venv/bin/python -c 'import time; from src.security import JWTParams, create_jwt, verify_jwt; token = create_jwt(JWTParams(exp=int(time.time()) + 60)); assert isinstance(verify_jwt(token).exp, int)'

RUN addgroup --system app \
    && adduser --system --group app \
    && chown -R app:app /src/honcho /app

USER app
WORKDIR /src/honcho

LABEL org.opencontainers.image.source="https://github.com/plastic-labs/honcho" \
      org.opencontainers.image.revision="${HONCHO_COMMIT}"

EXPOSE 8000

ENTRYPOINT ["sh", "docker/entrypoint.sh"]
