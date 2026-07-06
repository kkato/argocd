# Grafana Mimir

Prometheusのバックエンドストレージ（単一バイナリ `-target=all`、素マニフェスト）。オブジェクトストレージは使わず、openebs-hostpath上のfilesystemバックエンド(chartデフォルト)で運用する。

- イメージ: `grafana/mimir:3.1.2`
- ストレージ: `/data`配下を`blocks`/`alertmanager`/`ruler`のディレクトリで使い分け
- 永続化: `openebs-hostpath` 50Gi（実データを恒久保持するため）
- 単一replica運用のため各リング(ingester/store-gateway/alertmanager)の`replication_factor`は1に設定

## デプロイ後の確認

```bash
kubectl -n monitoring get pods -l app=mimir
kubectl -n monitoring logs statefulset/mimir

# Prometheusのremote_writeが成功しているか（kube-prometheus-stackデプロイ後）
kubectl -n monitoring port-forward svc/mimir 8080:8080 &
curl -s localhost:8080/prometheus/api/v1/query --data-urlencode 'query=up' | jq
```
