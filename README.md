# nig-podman

遺伝研（NIG）スパコンで rootless Podman を利用するためのセットアップ手順、サービス設定例、開発用テンプレート集。Docker に慣れたユーザーが Podman 環境にスムーズに移行できることを目的とする。

## 想定読者

- Docker の基本操作に慣れている NIG スパコンユーザー
- rootless Podman でのコンテナ運用を始めたい人
- Docker/Podman 両対応の開発環境を構築したい人

## リポジトリ構成

```
nig-podman/
├── docs/                         # ドキュメント
│   ├── introduction.md           #   Docker vs Podman の違い
│   ├── setup.md                  #   NIG 環境でのセットアップ
│   ├── uid-mapping.md            #   rootless UID マッピング解説
│   └── compose-tips.md           #   compose の Docker/Podman 両対応
└── examples/                     # 設定例・テンプレート
    ├── nginx/                    #   リバースプロキシ
    ├── postgres/                 #   PostgreSQL
    ├── elasticsearch/            #   Elasticsearch
    ├── python-uv/                #   Python + uv
    ├── python-pip/               #   Python + pip
    ├── typescript-bun/           #   TypeScript + Bun
    └── typescript-npm/           #   TypeScript + npm
```

## 基本コンセプト: `compose.override.podman.yml` パターン

ベースの `compose.yml` は Docker 互換で書き、Podman 固有の設定は `compose.override.podman.yml` に分離する:

```bash
# Docker 環境
docker compose up

# Podman 環境
podman-compose -f compose.yml -f compose.override.podman.yml up

# Podman で毎回 -f を省略したい場合
cp compose.override.podman.yml compose.override.yml
podman-compose up
```

詳細は [compose の Docker/Podman 両対応](./docs/compose-tips.md) を参照。

## ドキュメント

| ドキュメント | 内容 |
|---|---|
| [Docker と Podman の違い](./docs/introduction.md) | 基本的な違い、NIG で Podman を使う理由 |
| [NIG での Podman セットアップ](./docs/setup.md) | storage.conf, registries.conf の設定手順 |
| [rootless UID マッピング](./docs/uid-mapping.md) | UID マッピングの仕組み、keep-id オプション |
| [compose の Docker/Podman 両対応](./docs/compose-tips.md) | override パターン、パーミッション戦略 |

## 設定例・テンプレート

| 例 | 内容 | Podman 設定 |
|---|---|---|
| [nginx](./examples/nginx/) | リバースプロキシ | 追加設定不要（root で動作） |
| [PostgreSQL](./examples/postgres/) | DB サーバー | `keep-id:uid=999,gid=999` |
| [Elasticsearch](./examples/elasticsearch/) | 検索エンジン | `keep-id:uid=1000,gid=1000` |
| [python-uv](./examples/python-uv/) | Python + uv 開発環境 | `keep-id` |
| [python-pip](./examples/python-pip/) | Python + pip 開発環境 | `keep-id` |
| [typescript-bun](./examples/typescript-bun/) | TypeScript + Bun 開発環境 | `keep-id` |
| [typescript-npm](./examples/typescript-npm/) | TypeScript + npm 開発環境 | `keep-id` |

開発テンプレート（python-\*, typescript-\*）の構成:

- `Dockerfile` - 依存インストール + `chmod -R a+rwX` + 任意 UID 用ホーム作成
- `compose.yml` - ベース（named volume、Docker/Podman 共通）
- `compose.override.podman.yml` - Podman 用（`userns_mode: "keep-id"`、`:U` フラグ）
- `.dockerignore`

## Quick Start

### 1. Podman の初期設定（NIG スパコン）

```bash
mkdir -p ~/.config/containers
```

`~/.config/containers/storage.conf`:

```toml
[storage]
  driver = "overlay"
  graphroot = "/data1/{user}/storage"

[storage.options]
  mount_program = "/usr/bin/fuse-overlayfs"
```

`~/.config/containers/registries.conf`:

```toml
[registries.search]
registries = ['docker.io']
```

> `{user}` は自分のユーザー名に置き換える。詳細は [セットアップ手順](./docs/setup.md) を参照。

設定の確認:

```bash
podman info | grep -E 'graphRoot|graphDriverName'
```

### 2. 設定例の利用

```bash
cd examples/python-uv

# Docker 環境
docker compose up --build

# Podman 環境
podman-compose -f compose.yml -f compose.override.podman.yml up --build
```

env ファイルが必要な例（postgres, elasticsearch）:

```bash
cd examples/postgres
cp env.example .env
# .env を編集してパスワード等を設定
```

## 動作確認環境

| 環境 | OS | ツール |
|---|---|---|
| Docker (ローカル) | Ubuntu 22.04 | Docker 29.2.1 |
| Podman (NIG スパコン) | Ubuntu 24.04 | Podman 4.9.3, podman-compose 1.0.6 |

## License

[Apache License 2.0](./LICENSE)
