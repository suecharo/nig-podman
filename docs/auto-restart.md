# サーバー再起動後のコンテナ自動起動

rootless Podman のコンテナはユーザープロセスとして動作するため、サーバーが再起動するとコンテナは停止したままになる。`podman-restart.service` を有効にすることで、サーバー起動時にコンテナを自動的に再起動できる。

## 前提条件

Linger が有効であること。Linger が無効だとユーザーの systemd インスタンスがサーバー起動時に開始されず、`podman-restart.service` も実行されない。

Linger の設定方法は [NIG での Podman セットアップ](./setup.md#linger-の有効化) を参照。

## 手順

### 1. podman-restart.service の有効化

```bash
systemctl --user enable podman-restart.service
```

このサービスを有効にすると、サーバー起動時に `restart-policy=always` が設定されたコンテナが自動的に起動する。

### 2. compose.yml で restart policy を設定

自動起動したいサービスに `restart: always` を追加する:

```yaml
services:
  db:
    image: postgres:17
    restart: always
```

## 動作の仕組み

`podman-restart.service` はサーバー起動時に以下のコマンドを実行する:

```bash
podman start --all --filter restart-policy=always
```

このコマンドは restart policy が `always` に設定されたすべてのコンテナを起動する。`podman-compose up -d` で作成したコンテナも対象になる。

## 確認方法

`podman-restart.service` が有効か確認:

```bash
systemctl --user is-enabled podman-restart.service
```

`enabled` と表示されれば有効。

コンテナの restart policy を確認:

```bash
podman inspect --format '{{.HostConfig.RestartPolicy.Name}}' <container-name>
```

`always` と表示されれば自動起動の対象になる。
