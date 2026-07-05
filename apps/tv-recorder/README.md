# tv-recorder

TVer の見逃し配信を自動録画するレコーダー。アプリ本体は
[kkato/tv-recorder](https://github.com/kkato/tv-recorder) で管理し、
イメージは main への push で `ghcr.io/kkato/tv-recorder` に発行される。

- 予約 UI: 各ノードの NodePort 30080
- 予約ルール・録画状態は CNPG の PostgreSQL (`tv-recorder-db`)
- 録画データは共有 media PVC（apps/jellyfin 側で定義）の `/media/tv` に保存し、Jellyfin が配信する
