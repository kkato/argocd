# Loki

ログ集約（SingleBinary/monolithic mode）。MinIOをオブジェクトストレージのバックエンドに使用する。

- chart: [`grafana/loki`](https://github.com/grafana/loki) 7.0.0
- ストレージ: MinIOの`loki`バケット
- 永続化: `openebs-hostpath` 20Gi
- 保持期間: 30日（`limits_config.retention_period` + `compactor.retention_enabled`）
- 前段のnginx gatewayは無効化し、singleBinaryが直接3100番で応答する

## デプロイ前の手動準備

### S3認証情報

[addons/minio/README.md](../minio/README.md) の手順でMinIOの監視スタック用ユーザーを作成し、`secret/minio/monitoring`にVault登録しておく。Mimir/Loki/Tempoはこの1つの認証情報を共有する。個別の追加登録は不要。

## デプロイ後の確認

```bash
kubectl -n monitoring get pods -l app.kubernetes.io/name=loki

# Grafana Alloyから収集されたログがLokiに届いているか確認
kubectl -n monitoring port-forward svc/loki 3100:3100 &
curl -s -G localhost:3100/loki/api/v1/labels | jq
```
