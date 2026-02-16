# Docker と Podman の違い

## このリポジトリの目的

遺伝研（NIG）スパコンでは、セキュリティ上の理由から root 権限が付与されないため、rootful な Docker デーモンを利用できない。代わりに rootless Podman が利用可能である。

このリポジトリでは、Docker に慣れたユーザーが NIG 環境で Podman をスムーズに利用できるよう、以下を提供する:

- NIG 環境での Podman セットアップ手順
- よく使うサービス（nginx, PostgreSQL, Elasticsearch）の設定例
- 開発用テンプレート（Python, TypeScript）

## 想定読者

- Docker の基本的な操作（`docker run`, `docker compose up` 等）に慣れている
- NIG スパコンのアカウントを持っている
- rootless Podman は初めて、または UID マッピングの仕組みを理解したい

## Docker と Podman の基本的な違い

| 項目 | Docker | Podman |
|---|---|---|
| 実行モード | rootful（デーモンが root で動作） | rootless（ユーザー権限で動作） |
| デーモン | dockerd が常駐 | デーモンなし（fork/exec） |
| compose コマンド | `docker compose` | `podman-compose` |
| イメージ形式 | OCI 標準 | OCI 標準（互換） |
| デフォルトレジストリ | Docker Hub (`docker.io`) | なし（明示的に設定が必要） |

### rootful vs rootless

Docker はデーモン (`dockerd`) が root 権限で動作し、コンテナ内の root (UID 0) はホストの root (UID 0) にそのままマッピングされる。これはシンプルだが、コンテナからのブレイクアウト時にホストの root 権限を奪取されるリスクがある。

Podman は rootless モードで動作し、コンテナ内の root (UID 0) はホストの実行ユーザーの UID にマッピングされる。このため、コンテナからブレイクアウトしても得られるのはユーザー権限のみである。

### daemon vs daemonless

Docker は中央のデーモンプロセス (`dockerd`) が全コンテナを管理する。デーモンがクラッシュすると全コンテナに影響する。

Podman はデーモンを持たず、各コンテナは独立したプロセスとして実行される。systemd との統合も容易である。

### compose の互換性

Podman は `podman-compose` コマンドで Docker Compose ファイルを（ほぼ）そのまま利用できる。ただし Podman 固有のオプション（`userns_mode` 等）は別途 override ファイルで指定する必要がある。詳細は [compose の Docker/Podman 両対応](./compose-tips.md) を参照。

### イメージ互換性

Docker も Podman も OCI (Open Container Initiative) 標準に準拠している。Docker Hub のイメージは Podman でそのまま利用可能である。ただし、Podman はデフォルトで Docker Hub をレジストリとして検索しないため、`registries.conf` での設定が必要になる（[セットアップ手順](./setup.md) を参照）。

## NIG スパコンで Podman を使う理由

NIG スパコンでは一般ユーザーに root 権限が付与されないため、rootful Docker を直接利用できない。rootless Podman であればユーザー権限のみでコンテナを実行できるため、管理者への依頼なしにコンテナ環境を構築・運用できる。
