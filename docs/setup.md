# NIG での Podman セットアップ

NIG スパコン環境で rootless Podman を利用するための初期設定手順を説明する。

## 設定ファイルの配置場所

Podman の設定ファイルは `~/.config/containers/` に配置する:

```
~/.config/containers/
├── storage.conf
└── registries.conf
```

ディレクトリが存在しない場合は作成する:

```bash
mkdir -p ~/.config/containers
```

## storage.conf

NIG スパコンではホームディレクトリが Lustre ファイルシステム (`/lustre9/open/`) 上にあり、Podman のデフォルトストレージドライバーである overlay FS と相性が悪い。

コンテナイメージやレイヤーの保存先を、ローカル NVMe ディスク (`/data1/{user}/storage`) に設定することで、パフォーマンスと安定性を改善できる。

`~/.config/containers/storage.conf` を以下の内容で作成する:

```toml
[storage]
  driver = "overlay"
  graphroot = "/data1/{user}/storage"

[storage.options]
  mount_program = "/usr/bin/fuse-overlayfs"
```

> `{user}` は自分のユーザー名に置き換える。

### 各項目の説明

- `driver = "overlay"`: ストレージドライバーに overlay を使用する
- `graphroot = "/data1/{user}/storage"`: イメージやコンテナのデータをローカル NVMe に保存する。Lustre 上に置くと I/O 性能が著しく低下する
- `mount_program = "/usr/bin/fuse-overlayfs"`: rootless 環境では kernel overlay FS が使えないため、FUSE ベースの overlay を使用する

## registries.conf

Podman は Docker と異なり、デフォルトでは Docker Hub (`docker.io`) をレジストリとして検索しない。`docker.io` からイメージを pull するには明示的な設定が必要になる。

`~/.config/containers/registries.conf` を以下の内容で作成する:

```toml
[registries.search]
registries = ['docker.io']
```

この設定により、`podman pull python:3.12` のようにレジストリを省略した指定でも Docker Hub から pull できるようになる。

## 設定の確認

設定が正しく反映されているか確認する:

```bash
# Podman の全体情報を表示
podman info

# 確認すべき項目:
# - graphRoot が /data1/{user}/storage になっていること
# - graphDriverName が overlay になっていること
# - registries.search に docker.io が含まれていること
```

## トラブルシューティング

### `fuse-overlayfs` が見つからない

```
Error: fuse-overlayfs: command not found
```

`/usr/bin/fuse-overlayfs` のパスを確認する:

```bash
which fuse-overlayfs
```

パスが異なる場合は `storage.conf` の `mount_program` を修正する。

### ストレージの初期化

storage.conf を変更した場合、既存のストレージとの不整合が生じることがある。その場合はストレージをリセットする:

```bash
podman system reset
```

> このコマンドは全てのコンテナ・イメージ・ボリュームを削除する。実行前に必要なデータがないか確認すること。
