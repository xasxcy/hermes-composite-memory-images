ARG PYTHON_BASE_IMAGE
FROM ${PYTHON_BASE_IMAGE}

ARG MEM0_REPOSITORY=https://github.com/mem0ai/mem0.git
ARG MEM0_COMMIT=001c235229be8795e3834520467bd0d661ed8f34

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

RUN apt-get update \
    && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
RUN git clone "${MEM0_REPOSITORY}" mem0 \
    && cd mem0 \
    && git checkout --detach "${MEM0_COMMIT}" \
    && test "$(git rev-parse HEAD)" = "${MEM0_COMMIT}"

RUN PIP_DEFAULT_TIMEOUT=120 PIP_RETRIES=10 python -m pip install -r /src/mem0/server/requirements.txt \
    && PIP_DEFAULT_TIMEOUT=120 PIP_RETRIES=10 python -m pip install "psycopg[binary]" \
    && PIP_DEFAULT_TIMEOUT=120 PIP_RETRIES=10 python -m pip install /src/mem0 \
    && python -c "import mem0; assert mem0.__version__ == '2.0.18', mem0.__version__" \
    && python -c "import psycopg; assert psycopg.pq.__impl__ in ('binary', 'c'), psycopg.pq.__impl__"

WORKDIR /app
RUN cp -a /src/mem0/server/. /app/ \
    && rm -rf /src/mem0/.git

LABEL org.opencontainers.image.source="https://github.com/mem0ai/mem0" \
      org.opencontainers.image.revision="${MEM0_COMMIT}"

EXPOSE 8000

CMD ["sh", "-c", "alembic upgrade head && exec uvicorn main:app --host 0.0.0.0 --port 8000"]
