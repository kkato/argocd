# Grafana Mimir

Prometheusのバックエンドストレージ（単一バイナリ `-target=all`、素マニフェスト）。MinIOをオブジェクトストレージのバックエンドに使用する。

- イメージ: `grafana/mimir:3.1.2`
- ストレージ: MinIOの`mimir`バケットを`blocks`/`alertmanager`/`ruler`のprefixで使い分け
- 永続化: `openebs-hostpath` 20Gi（ローカルWAL/TSDBブロック用。実データはS3へshipする）
- 単一replica運用のため各リング(ingester/store-gateway/alertmanager)の`replication_factor`は1に設定

## デプロイ前の手動準備

### S3認証情報

[addons/minio/README.md](../minio/README.md) の手順でMinIOの監視スタック用ユーザーを作成し、`secret/minio/monitoring`にVault登録しておく。Mimir/Loki/Tempoはこの1つの認証情報を共有する。個別の追加登録は不要。

## デプロイ後の確認

```bash
kubectl -n monitoring get pods -l app=mimir
kubectl -n monitoring logs statefulset/mimir

# Prometheusのremote_writeが成功しているか（kube-prometheus-stackデプロイ後）
kubectl -n monitoring port-forward svc/mimir 8080:8080 &
curl -s localhost:8080/prometheus/api/v1/query --data-urlencode 'query=up' | jq
```
