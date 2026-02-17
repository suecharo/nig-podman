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

## 細かい挙動差分

Docker からの移行時に気づきにくい、Podman 固有の挙動をまとめる。

- **`working_dir` の自動作成**: Docker は compose の `working_dir` や `docker run -w` で指定したディレクトリが存在しない場合に自動作成するが、Podman はエラーになる。Dockerfile の `WORKDIR` はビルド時に自動作成されるため問題ない。ランタイムで `working_dir` を使う場合は Dockerfile に `WORKDIR` を書いてイメージ内にディレクトリを作成しておく

  ```dockerfile
  WORKDIR /app
  ```

- **バインドマウント元ディレクトリの自動作成**: Docker はホスト側のマウント元ディレクトリが存在しない場合に root で自動作成するが、rootless Podman は自動作成しない。事前にホスト側でディレクトリを作成しておく

  ```bash
  mkdir -p ./data
  ```

- **イメージのデフォルトユーザー確認**: イメージが非 root ユーザーで動作する場合、`keep-id` の設定が必要になることがある。以下のコマンドでイメージ定義上のユーザーを確認できる

  ```bash
  podman image inspect --format '{{json .Config.User}}' <image-name>
  ```

  空文字列または `"0"` なら定義上は root で動作する。`"472"` (Grafana) のように非 root が設定されている場合は [rootless UID マッピング](./uid-mapping.md) を参照して `keep-id` の設定を検討する

  ただし PostgreSQL のように、`Config.User` は空（root）でも entrypoint 内で `gosu` 等によりプロセスを別ユーザーに切り替えるイメージもある。実際の実行ユーザーを確認するにはコンテナ起動後に `ps` で確認する:

  ```bash
  podman exec <container-name> ps aux | head -5
  ```

  > `podman exec ... id` ではなく `ps aux` を使う理由: `podman exec` は `Config.User`（この場合 root）として新しいプロセスを起動するため、`id` は root を返す。entrypoint 内で `gosu` により切り替わった PID 1 の実行ユーザーは `ps` でないと確認できない。
