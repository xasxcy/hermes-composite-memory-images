ARG PYTHON_BASE_IMAGE
ARG UV_IMAGE
FROM ${UV_IMAGE} AS uv
FROM ${PYTHON_BASE_IMAGE}

ARG HONCHO_REPOSITORY=https://github.com/plastic-labs/honcho.git
ARG HONCHO_COMMIT=0cd37ddbd88f51cdac7c660d7372f4b2fd40a4b3

COPY --from=uv /uv /bin/uv

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UV_LINK_MODE=copy \
    UV_PROJECT_ENVIRONMENT=/app/.venv \
    TIKTOKEN_CACHE_DIR=/app/tiktoken-cache \
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
RUN mkdir -p "${TIKTOKEN_CACHE_DIR}"
RUN uv sync --frozen --no-group dev --python /usr/local/bin/python \
    && test -x /app/.venv/bin/python \
    && /app/.venv/bin/python -c 'import src' \
    && AUTH_JWT_SECRET=01234567890123456789012345678901 /app/.venv/bin/python -c 'import datetime as d, unittest; from src.exceptions import AuthenticationException as E; from src.security import JWTParams as P, create_jwt as c, verify_jwt as v; n=d.datetime.now(d.UTC); v(c(P(ad=True, exp=n+d.timedelta(seconds=60)))); unittest.TestCase().assertRaisesRegex(E, "expired", v, c(P(ad=True, exp=n-d.timedelta(seconds=60))))'

# 构建期预置 tiktoken 词表（o200k_base 在 import 期使用，cl100k_base 为 embedding_client 的编码/兜底）。
# 缓存由 tiktoken 自己写入并校验内容哈希；下面的 --network=none 断言保证运行期无需联网（ADR-162）。
RUN /app/.venv/bin/python -c 'import tiktoken; tiktoken.get_encoding("o200k_base"); tiktoken.get_encoding("cl100k_base")' \
    && ls "${TIKTOKEN_CACHE_DIR}" | wc -l
RUN --network=none /app/.venv/bin/python -c 'import src.utils.tokens, src.schemas.api, tiktoken; tiktoken.get_encoding("o200k_base"); tiktoken.get_encoding("cl100k_base")'

RUN addgroup --system app \
    && adduser --system --group app \
    && chown -R app:app /src/honcho /app

USER app
WORKDIR /src/honcho

LABEL org.opencontainers.image.source="https://github.com/plastic-labs/honcho" \
      org.opencontainers.image.revision="${HONCHO_COMMIT}"

EXPOSE 8000

ENTRYPOINT ["sh", "docker/entrypoint.sh"]
