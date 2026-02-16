# rootless UID マッピング

rootless Podman を理解する上で最も重要な概念が UID マッピングである。Docker と Podman でコンテナ内の UID がホスト上でどのように見えるかが大きく異なる。

## rootful Docker の UID マッピング

Docker ではデーモンが root で動作するため、コンテナ内の UID はそのままホストの UID に対応する:

```
Container UID    Host UID
+-----------+    +-----------+
| 0 (root)  | -> | 0 (root)  |
| 1         | -> | 1         |
| 1000      | -> | 1000      |
+-----------+    +-----------+
```

シンプルだが、コンテナ内の root がホストの root と同一のため、コンテナブレイクアウト時のリスクが大きい。

## rootless Podman の UID マッピング

rootless Podman では、コンテナ内の UID がホスト上の異なる UID にマッピングされる。例えば、ホスト UID が 4589 のユーザーの場合:

```
Container UID    Host UID
+-----------+    +-------------+
| 0 (root)  | -> | 4589        |  (your UID)
| 1         | -> | 458900000   |  (subuid start)
| 2         | -> | 458900001   |
| ...       |    | ...         |
| 65536     | -> | 458965535   |  (subuid end)
+-----------+    +-------------+
```

- コンテナ内の root (UID 0) はホストの自分自身の UID (4589) にマッピングされる
- コンテナ内の UID 1-65536 は `/etc/subuid` で割り当てられた範囲にマッピングされる

### `/etc/subuid` と `/etc/subgid`

サブ UID/GID の割り当ては `/etc/subuid` と `/etc/subgid` に記載されている:

```
# /etc/subuid の例
username:458900000:65536
```

この例では、`username` に UID 458900000 から 65536 個の UID（458900000-458965535）が割り当てられている。

### マッピングの確認

現在のユーザー名前空間でのマッピングを確認する:

```bash
podman unshare cat /proc/self/uid_map
```

## バインドマウントへの影響

UID マッピングはバインドマウント時のファイル所有権に直接影響する。

### コンテナ内 root でファイルを作成した場合

コンテナ内の root (UID 0) = ホストの自分の UID なので、ホスト上で見ても自分の所有ファイルになる。**これは便利な特性である。**

```bash
# コンテナ内で root としてファイルを作成
podman run --rm -v ./data:/data alpine touch /data/test.txt

# ホスト上で確認 -> 自分の UID で所有されている
ls -la ./data/test.txt
# -rw-r--r-- 1 username username 0 ... test.txt
```

### コンテナ内の非 root ユーザーでファイルを作成した場合

コンテナ内の非 root UID は subuid 範囲にマッピングされるため、ホスト上では見慣れない大きな UID で所有される。

```bash
# コンテナ内で UID 1000 としてファイルを作成
podman run --rm -v ./data:/data --user 1000:1000 alpine touch /data/test.txt

# ホスト上で確認 -> subuid 範囲の UID で所有されている
ls -la ./data/test.txt
# -rw-r--r-- 1 458900999 458900999 0 ... test.txt
```

**このファイルはホスト上の自分のユーザーでは読み書きできない場合がある。** これが rootless Podman でよく遭遇するパーミッション問題の原因である。

## `keep-id` オプション

`keep-id` は、ホストの UID をコンテナ内でもそのまま使うためのオプションである。

### 基本的な `keep-id`

```yaml
# compose.override.podman.yml
services:
  app:
    userns_mode: "keep-id"
```

この設定では、ホストの UID (例: 4589) がコンテナ内でも UID 4589 として見える。コンテナ内でファイルを作成すると、ホスト上でも同じ UID で所有されるため、パーミッション問題が解消される。

### 特定 UID へのマッピング

PostgreSQL (UID 999) や Elasticsearch (UID 1000) のように、特定の UID で動作するサービスでは、ホスト UID をその UID にマッピングする:

```yaml
# PostgreSQL の例
services:
  db:
    userns_mode: "keep-id:uid=999,gid=999"
```

この設定では、ホストの UID (例: 4589) がコンテナ内では UID 999 (postgres ユーザー) として見える。PostgreSQL がデータファイルを作成すると、ホスト上では自分の UID (4589) で所有される。

## まとめ

| 状況 | コンテナ内 UID | ホスト UID | 備考 |
|---|---|---|---|
| rootful Docker | 0 (root) | 0 (root) | セキュリティリスクあり |
| rootless Podman (デフォルト) | 0 (root) | 自分の UID | 便利 |
| rootless Podman (デフォルト) | 1000 | subuid 範囲 | パーミッション問題の原因 |
| rootless Podman (`keep-id`) | 自分の UID | 自分の UID | 開発向け |
| rootless Podman (`keep-id:uid=999`) | 999 | 自分の UID | DB 等のサービス向け |
