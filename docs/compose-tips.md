# compose の Docker/Podman 両対応

Docker と Podman の両方で動く compose 環境を構築するためのパターンと Tips を説明する。

## `compose.override.podman.yml` パターン

ベースの `compose.yml` は Docker 互換で書き、Podman 固有の設定（`userns_mode` 等）は `compose.override.podman.yml` に分離する。

```
project/
├── compose.yml                      # Docker 互換（ベース）
└── compose.override.podman.yml      # Podman 固有の設定
```

### 使い分け

Docker 環境:

```bash
docker compose up
```

Podman 環境:

```bash
podman-compose -f compose.yml -f compose.override.podman.yml up
```

Podman 環境で毎回 `-f` を指定したくない場合は、override ファイルをコピーする:

```bash
cp compose.override.podman.yml compose.override.yml
podman-compose up
```

`compose.override.yml` は Docker Compose / podman-compose が自動的に読み込むファイル名であるため、コピーするだけで追加の指定なしに Podman 設定が適用される。

> `compose.override.yml` は `.gitignore` に追加しておく。各環境で使う override ファイルが異なるため、リポジトリには含めない。

### ファイルの書き方

`compose.yml` (ベース):

```yaml
services:
  app:
    build: .
    volumes:
      - .:/app:rw
      - node_modules:/app/node_modules:rw
    init: true

volumes:
  node_modules: {}
```

`compose.override.podman.yml` (Podman 固有):

```yaml
services:
  app:
    userns_mode: "keep-id"
    volumes:
      - .:/app:rw
      - node_modules:/app/node_modules:rw,U
```

ポイント:

- ベースの `compose.yml` は Docker/Podman 共通で使える最小構成にする
- `userns_mode: "keep-id"` で UID マッピングを設定
- named volume に `:U` フラグを追加（volume の所有者をコンテナ内ユーザーに自動修正）

## パーミッション戦略

Docker と Podman の両方でパーミッション問題なく動く Dockerfile の書き方。

### 1. 依存ディレクトリに `chmod -R a+rwX`

`.venv` や `node_modules` など、named volume としてマウントされるディレクトリは任意の UID から書き込めるようにする:

```dockerfile
RUN npm ci && \
    chmod -R a+rwX node_modules
```

named volume は初回作成時にイメージ内のパーミッションを引き継ぐため、ここで設定した権限が volume にも反映される。

### 2. 任意 UID 用のホームディレクトリを作成

コンテナが任意の UID で実行される場合、その UID のホームディレクトリが存在しないとツールが正常に動作しないことがある:

```dockerfile
ENV HOME=/home/app
RUN mkdir -p /home/app && chmod 777 /home/app
```

### 3. Podman 環境では `userns_mode` を使用

```yaml
# compose.override.podman.yml
services:
  app:
    userns_mode: "keep-id"
```

ホスト UID をコンテナ内にそのままマッピングする。`user` ディレクティブは不要（keep-id で自動的にマッピングされる）。

## named volume の活用

### なぜ named volume を使うのか

`.venv` や `node_modules` のような依存ディレクトリをバインドマウントすると、UID マッピングの違いによりパーミッション問題が発生しやすい。named volume はコンテナ内部で管理されるため、ホストの UID マッピングの影響を受けない。

```yaml
services:
  app:
    volumes:
      - .:/app:rw                          # source code (bind mount)
      - node_modules:/app/node_modules:rw  # deps (named volume)

volumes:
  node_modules: {}
```

### Podman での `:U` フラグ

Podman の named volume に `:U` フラグを指定すると、volume 内のファイルの所有者がコンテナ内のユーザーに自動修正される:

```yaml
# compose.override.podman.yml
services:
  app:
    volumes:
      - .:/app:rw
      - node_modules:/app/node_modules:rw,U
```

> `:U` は Podman 固有のフラグであるため、Docker 用の `compose.yml` には記載せず、Podman 用の override ファイルにのみ記載する。

## その他の Tips

### `init: true` を常に指定する

コンテナの PID 1 プロセスが init でない場合、シグナルが正しく処理されず `SIGTERM` でコンテナが停止しないことがある。`init: true` を指定すると tini が PID 1 として動作し、シグナルを適切に転送する:

```yaml
services:
  app:
    init: true
```

### 環境変数ファイルの管理

環境変数は `env.example` をテンプレートとしてリポジトリに含め、利用時に `.env` にコピーする:

```yaml
# compose.yml
services:
  db:
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:?POSTGRES_PASSWORD is required}
```

```bash
cp env.example .env
# .env を編集して値を設定
```

`.env` は Docker Compose / podman-compose が自動的に読み込むため、起動時のオプション指定は不要である。

`.env` は秘密情報を含むため `.gitignore` に追加し、リポジトリには含めない。
