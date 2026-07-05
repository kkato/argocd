# radio-recorder

radiko のタイムフリーを自動録音するレコーダー。アプリ本体は
[kkato/radio-recorder](https://github.com/kkato/radio-recorder) で管理し、
イメージは main への push で `ghcr.io/kkato/radio-recorder` に発行される。

- 予約 UI: 各ノードの NodePort 30081
- 予約ルール・録音状態は CNPG の PostgreSQL (`radio-recorder-db`)
- 録音データは共有 media PVC（apps/jellyfin 側で定義）の `/media/radio` に保存し、Jellyfin が配信する
