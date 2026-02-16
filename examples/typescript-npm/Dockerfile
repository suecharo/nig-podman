FROM node:24-bookworm

WORKDIR /app

COPY package.json package-lock.json ./

# chmod: named volume inherits image permissions on first creation;
#        make writable so arbitrary UID (dev) can run `npm install`.
RUN npm ci && \
    mkdir -p node_modules && \
    chmod -R a+rwX node_modules

COPY . .

# Writable home for arbitrary UID (dev containers use user: UID:GID).
ENV HOME=/home/app
RUN mkdir -p /home/app && chmod 777 /home/app

CMD ["sleep", "infinity"]
