FROM python:3.12-bookworm

WORKDIR /app

COPY requirements.txt ./

# chmod: named volume inherits image permissions on first creation;
#        make writable so arbitrary UID (dev) can run `pip install`.
RUN python -m venv .venv && \
    .venv/bin/pip install --no-cache-dir -r requirements.txt && \
    chmod -R a+rwX .venv

COPY . .

# Writable home for arbitrary UID (dev containers use user: UID:GID).
ENV HOME=/home/app
RUN mkdir -p /home/app && chmod 777 /home/app

ENV PATH="/app/.venv/bin:$PATH"

CMD ["sleep", "infinity"]
