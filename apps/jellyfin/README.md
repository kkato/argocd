# jellyfin

tv-recorder / radio-recorder が保存した番組を配信するメディアサーバー。
共有の media PVC はここで定義し、レコーダーは RW、Jellyfin は RO でマウントする。

- 視聴: 各ノードの NodePort 30096
- 初期設定: 管理者作成後、ライブラリを2つ追加する
  （種類「番組」で `/media/tv` と `/media/radio`、NFO リーダー有効）
- media PVC は openebs-hostpath（ノードローカル）。RWO のため media を使う
  Pod は同一ノードに載る。NAS 導入時は storageClassName を差し替える
