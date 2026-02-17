# NIG での Podman セットアップ

NIG スパコン環境で rootless Podman を利用するための初期設定手順。

## 前提条件: subuid / subgid

- rootless Podman は `/etc/subuid` と `/etc/subgid` を使い、コンテナ内 UID をホスト UID にマッピングする（詳細は [rootless UID マッピング](./uid-mapping.md)）
- これらはシステムファイルのため設定に root 権限が必要
- NIG スパコンでは管理者が設定済みのため、ユーザー側の作業は不要

設定状況の確認:

```bash
grep $(whoami) /etc/subuid /etc/subgid
```

出力例:

```
/etc/subuid:username:458900000:65536
/etc/subgid:username:458900000:65536
```

この出力が得られない場合は、管理者に subuid/subgid の割り当てを依頼する。

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

- NIG のホームは Lustre FS (`/lustre9/open/`) 上にあり、overlay FS と相性が悪い
- ローカル NVMe (`/data1/{user}/storage`) に保存先を変更することでパフォーマンスが改善する

`~/.config/containers/storage.conf` を以下の内容で作成する:

```toml
[storage]
  driver = "overlay"
  graphroot = "/data1/{user}/storage"

[storage.options]
  mount_program = "/usr/bin/fuse-overlayfs"
```

> `{user}` は自分のユーザー名に置き換える。

各項目:

- `driver = "overlay"`: ストレージドライバーに overlay を使用
- `graphroot = "/data1/{user}/storage"`: イメージやコンテナのデータをローカル NVMe に保存。Lustre 上だと I/O 性能が著しく低下する
- `mount_program = "/usr/bin/fuse-overlayfs"`: rootless 環境では kernel overlay FS が使えないため FUSE ベースの overlay を使用

## registries.conf

Podman はデフォルトで Docker Hub (`docker.io`) を検索しない。`docker.io` からイメージを pull するには明示的な設定が必要。

`~/.config/containers/registries.conf` を以下の内容で作成する:

```toml
[registries.search]
registries = ['docker.io']
```

この設定により、`podman pull python:3.12` のようにレジストリを省略した指定でも Docker Hub から pull できるようになる。

## Linger の有効化

rootless Podman のコンテナはユーザーのプロセスとして動作する。デフォルトでは SSH セッションを切断するとユーザーのプロセスが終了し、`podman-compose up -d` で起動したバックグラウンドコンテナも停止する。

Linger を有効にすると、ログアウト後もユーザーのプロセスが維持される:

```bash
loginctl enable-linger
```

確認:

```bash
loginctl show-user $(whoami) --property=Linger
```

`Linger=yes` と表示されれば有効。この設定は永続化されるため、一度実行すれば再設定は不要。

> Linger はログアウト後のプロセス維持のみを行う。サーバー再起動後にコンテナを自動起動するには追加の設定が必要。詳細は [サーバー再起動後のコンテナ自動起動](./auto-restart.md) を参照。

## 設定の確認

```bash
podman info
```

確認すべき項目:

- `graphRoot` が `/data1/{user}/storage` になっている
- `graphDriverName` が `overlay` になっている
- `registries.search` に `docker.io` が含まれている

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
