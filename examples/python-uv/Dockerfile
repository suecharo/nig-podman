FROM python:3.12-bookworm

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

COPY pyproject.toml uv.lock ./

# chmod: named volume inherits image permissions on first creation;
#        make writable so arbitrary UID (dev) can run `uv run` / `uv sync`.
RUN uv sync && \
    chmod -R a+rwX .venv

COPY . .

# Writable home for arbitrary UID (dev containers use user: UID:GID).
ENV HOME=/home/app
RUN mkdir -p /home/app && chmod 777 /home/app

ENV PATH="/app/.venv/bin:$PATH"

CMD ["sleep", "infinity"]
