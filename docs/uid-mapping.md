# rootless UID マッピング

rootless Podman を理解する上で最も重要な概念が UID マッピングである。

## rootful Docker の UID マッピング

コンテナ内の UID はそのままホストの UID に対応する:

```
Container UID    Host UID
+-----------+    +-----------+
| 0 (root)  | -> | 0 (root)  |
| 1         | -> | 1         |
| 1000      | -> | 1000      |
+-----------+    +-----------+
```

コンテナ内の root がホストの root と同一のため、ブレイクアウト時のリスクが大きい。

## rootless Podman の UID マッピング

コンテナ内の UID がホスト上の異なる UID にマッピングされる。ホスト UID が 4589 のユーザーの場合:

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

- コンテナ内の root (UID 0) → ホストの自分の UID (4589) にマッピング
- コンテナ内の UID 1-65536 → `/etc/subuid` で割り当てられた範囲にマッピング

`/etc/subuid` と `/etc/subgid` の例:

```
# /etc/subuid
username:458900000:65536
```

`username` に UID 458900000 から 65536 個の UID（458900000-458965535）が割り当てられている。

マッピングの確認:

```bash
podman unshare cat /proc/self/uid_map
```

## バインドマウントへの影響

UID マッピングはバインドマウント時のファイル所有権に直接影響する:

- **コンテナ内 root でファイルを作成** → ホストでも自分の UID で所有される（便利）
- **コンテナ内の非 root UID でファイルを作成** → subuid 範囲の大きな UID になり、ホストから読み書きできない場合がある（パーミッション問題の原因）

コンテナ内 root の場合:

```bash
# コンテナ内で root としてファイルを作成
podman run --rm -v ./data:/data alpine touch /data/test.txt

# ホスト上で確認 -> 自分の UID で所有されている
ls -la ./data/test.txt
# -rw-r--r-- 1 username username 0 ... test.txt
```

コンテナ内の非 root UID の場合:

```bash
# コンテナ内で UID 1000 としてファイルを作成
podman run --rm -v ./data:/data --user 1000:1000 alpine touch /data/test.txt

# ホスト上で確認 -> subuid 範囲の UID で所有されている
ls -la ./data/test.txt
# -rw-r--r-- 1 458900999 458900999 0 ... test.txt
```

## `keep-id` オプション

ホストの UID をコンテナ内でもそのまま使うためのオプション。

基本的な `keep-id`:

```yaml
# compose.override.podman.yml
services:
  app:
    userns_mode: "keep-id"
```

ホスト UID がコンテナ内でもそのまま見え、ファイル所有権の不一致が解消される。

特定 UID へのマッピング:

```yaml
# PostgreSQL の例
services:
  db:
    userns_mode: "keep-id:uid=999,gid=999"
```

PostgreSQL (UID 999) や Elasticsearch (UID 1000) のように特定 UID で動作するサービスで使用する。ホスト UID (例: 4589) がコンテナ内では UID 999 として見え、データファイルはホスト上で自分の UID で所有される。

## まとめ

| 状況 | コンテナ内 UID | ホスト UID | 備考 |
|---|---|---|---|
| rootful Docker | 0 (root) | 0 (root) | セキュリティリスクあり |
| rootless Podman (デフォルト) | 0 (root) | 自分の UID | 便利 |
| rootless Podman (デフォルト) | 1000 | subuid 範囲 | パーミッション問題の原因 |
| rootless Podman (`keep-id`) | 自分の UID | 自分の UID | 開発向け |
| rootless Podman (`keep-id:uid=999`) | 999 | 自分の UID | DB 等のサービス向け |
