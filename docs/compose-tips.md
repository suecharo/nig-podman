# compose の Docker/Podman 両対応

Docker と Podman の両方で動く compose 環境を構築するためのパターンと Tips。

## `compose.override.podman.yml` パターン

ベースの `compose.yml` は Docker 互換で書き、Podman 固有の設定（`userns_mode` 等）は `compose.override.podman.yml` に分離する。

```
project/
├── compose.yml                      # Docker 互換（ベース）
└── compose.override.podman.yml      # Podman 固有の設定
```

Docker 環境:

```bash
docker compose up
```

Podman 環境:

```bash
podman-compose -f compose.yml -f compose.override.podman.yml up
```

毎回 `-f` を指定したくない場合は override ファイルをコピーする:

```bash
cp compose.override.podman.yml compose.override.yml
podman-compose up
```

`compose.override.yml` は Docker Compose / podman-compose が自動的に読み込むファイル名のため、コピーするだけで Podman 設定が適用される。

> `compose.override.yml` は `.gitignore` に追加しておく。各環境で使う override ファイルが異なるため、リポジトリには含めない。

## ファイルの書き方

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

Docker と Podman の両方でパーミッション問題なく動く Dockerfile の書き方:

- 依存ディレクトリに `chmod -R a+rwX` を適用する。named volume は初回作成時にイメージ内のパーミッションを引き継ぐため、ここで設定した権限が volume にも反映される

  ```dockerfile
  RUN npm ci && \
      chmod -R a+rwX node_modules
  ```

- 任意 UID 用のホームディレクトリを作成する。コンテナが任意の UID で実行される場合、ホームがないとツールが正常に動作しないことがある

  ```dockerfile
  ENV HOME=/home/app
  RUN mkdir -p /home/app && chmod 777 /home/app
  ```

- Podman 環境では `userns_mode` を使用する。ホスト UID をコンテナ内にそのままマッピングし、`user` ディレクティブは不要

  ```yaml
  # compose.override.podman.yml
  services:
    app:
      userns_mode: "keep-id"
  ```

## named volume の活用

- `.venv` や `node_modules` をバインドマウントすると UID マッピングの違いでパーミッション問題が発生しやすい
- named volume はコンテナ内部で管理されるため、ホストの UID マッピングの影響を受けない

```yaml
services:
  app:
    volumes:
      - .:/app:rw                          # source code (bind mount)
      - node_modules:/app/node_modules:rw  # deps (named volume)

volumes:
  node_modules: {}
```

Podman の named volume に `:U` フラグを指定すると、volume 内のファイル所有者がコンテナ内ユーザーに自動修正される:

```yaml
# compose.override.podman.yml
services:
  app:
    volumes:
      - .:/app:rw
      - node_modules:/app/node_modules:rw,U
```

> `:U` は Podman 固有のフラグのため、Docker 用の `compose.yml` には記載せず Podman 用の override ファイルにのみ記載する。

## その他の Tips

- **`init: true` を常に指定する**: PID 1 が init でない場合、`SIGTERM` でコンテナが停止しないことがある。`init: true` で tini が PID 1 として動作し、シグナルを適切に転送する

  ```yaml
  services:
    app:
      init: true
  ```

- **環境変数ファイルの管理**: `env.example` をテンプレートとしてリポジトリに含め、利用時に `.env` にコピーする。`.env` は Docker Compose / podman-compose が自動で読み込むため起動時のオプション指定は不要。秘密情報を含むため `.gitignore` に追加する

  ```bash
  cp env.example .env
  # .env を編集して値を設定
  ```

- **特権ポートの制限**: rootless では 1024 未満のポートにバインドできない。ホスト側のポートを 1024 以上に変更してマッピングする

  ```yaml
  services:
    web:
      ports:
        - "127.0.0.1:8080:80"
  ```

- **バインドアドレスの明示**: ポート指定でホストアドレスを省略すると `0.0.0.0`（全インターフェース）にバインドされる。NIG スパコンのようなマルチユーザー環境ではセキュリティ上 `127.0.0.1` を明示する

  ```yaml
  ports:
    # NG: 全インターフェースに公開される
    - "8080:80"
    # OK: localhost のみ
    - "127.0.0.1:8080:80"
  ```
