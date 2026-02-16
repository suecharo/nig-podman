FROM docker.io/oven/bun:latest

WORKDIR /app

COPY package.json bun.lock ./

# chmod: named volume inherits image permissions on first creation;
#        make writable so arbitrary UID (dev) can run `bun install`.
RUN bun install --frozen-lockfile && \
    chmod -R a+rwX node_modules

COPY . .

# Writable home for arbitrary UID (dev containers use user: UID:GID).
ENV HOME=/home/app
RUN mkdir -p /home/app && chmod 777 /home/app

CMD ["sleep", "infinity"]
