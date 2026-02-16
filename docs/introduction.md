# Docker と Podman の違い

NIG スパコンでは一般ユーザーに root 権限が付与されないため、rootful Docker を利用できない。rootless Podman であればユーザー権限のみで、管理者への依頼なしにコンテナ環境を構築・運用できる。

## 基本的な違い

| 項目 | Docker | Podman |
|---|---|---|
| 実行モード | rootful（デーモンが root で動作） | rootless（ユーザー権限で動作） |
| デーモン | dockerd が常駐 | デーモンなし（fork/exec） |
| compose コマンド | `docker compose` | `podman-compose` |
| イメージ形式 | OCI 標準 | OCI 標準（互換） |
| デフォルトレジストリ | Docker Hub (`docker.io`) | なし（明示的に設定が必要） |

- **rootful vs rootless**: Docker のコンテナ root = ホスト root（ブレイクアウト時にリスク大）。Podman のコンテナ root = ホストの実行ユーザー（リスク小）
- **daemon vs daemonless**: Docker は中央のデーモン (`dockerd`) が全コンテナを管理（クラッシュで全滅）。Podman は各コンテナが独立プロセスで、systemd との統合も容易
- **compose の互換性**: `podman-compose` で Docker Compose ファイルをほぼそのまま利用可能。ただし Podman 固有のオプション（`userns_mode` 等）は override ファイルで指定する（[詳細](./compose-tips.md)）
- **イメージ互換性**: Docker も Podman も OCI 標準準拠のため Docker Hub のイメージはそのまま利用可能。ただし Podman はデフォルトで Docker Hub を検索しないため `registries.conf` の設定が必要（[セットアップ](./setup.md)）
