# podman.sock と Docker CLI の連携

Podman は Docker 互換の API ソケット (`podman.sock`) を提供しており、Docker CLI からそのまま Podman を操作できる。

## podman.sock の概要

Docker では `dockerd` デーモンが `/var/run/docker.sock` を公開し、`docker` CLI やツールはこのソケットを通じてコンテナを操作する。Podman はデーモンレスだが、互換性のために同等の API ソケットを提供する仕組みがある。

rootless 環境でのソケットパス:

```
$XDG_RUNTIME_DIR/podman/podman.sock
```

NIG スパコンでは systemd の `podman.socket` ユニットが有効化されており、ソケットは自動的にリッスン状態になっている:

```bash
systemctl --user status podman.socket
```

ソケットファイルの存在確認:

```bash
ls -la $XDG_RUNTIME_DIR/podman/podman.sock
```

> `podman.socket` が有効でない環境では `podman system service --time=0 unix://$XDG_RUNTIME_DIR/podman/podman.sock &` で手動起動できる。

## ホストからの Docker CLI 利用

NIG スパコンには Docker CLI (`docker.io` パッケージ) がインストールされている。`DOCKER_HOST` 環境変数で Podman のソケットを指定すると、Docker CLI から Podman を操作できる:

```bash
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
docker ps
docker images
```

コマンド単位で指定する場合は `-H` オプションを使う:

```bash
docker -H unix://$XDG_RUNTIME_DIR/podman/podman.sock ps
```

> `DOCKER_HOST` を設定しない場合、Docker CLI はデフォルトの `/var/run/docker.sock` に接続しようとして権限エラーになる。

## コンテナ内からの利用 (sibling パターン)

CI/CD やビルドツールのコンテナ内から、ホストの Podman を操作したい場合がある（Docker-in-Docker ならぬ Docker-outside-of-Docker / sibling パターン）。

コンテナ内にホストの Podman ソケットをマウントし、Docker CLI で操作する:

```yaml
services:
  builder:
    image: docker:cli
    volumes:
      - ${XDG_RUNTIME_DIR}/podman/podman.sock:/var/run/docker.sock
```

`docker:cli` イメージはデフォルトで `/var/run/docker.sock` に接続するため、`DOCKER_HOST` の設定は不要。

Podman CLI をコンテナ内で使う場合は、ソケットに加えてストレージ設定やデータディレクトリのマウントも必要になるため煩雑になる。特別な理由がなければコンテナ内からは Docker CLI の利用を推奨する。

## まとめ

| 利用場所 | CLI | 推奨度 | 備考 |
|---|---|---|---|
| ホスト | Podman CLI | 推奨 | ネイティブ、追加設定不要 |
| ホスト | Docker CLI | 可 | `DOCKER_HOST` の設定が必要 |
| コンテナ内 | Docker CLI | 推奨 | ソケットのマウントのみで動作 |
| コンテナ内 | Podman CLI | 非推奨 | ストレージ設定等の追加マウントが必要 |
