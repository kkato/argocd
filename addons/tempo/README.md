# Tempo

分散トレーシング（monolithic mode）。オブジェクトストレージは使わず、chartデフォルトのlocalバックエンド(openebs-hostpath)で運用する。

- chart: [`tempo`](https://github.com/grafana-community/helm-charts) 2.2.3
- ストレージ: `openebs-hostpath`上のlocalバックエンド(chartデフォルト)
- 永続化: `openebs-hostpath` 10Gi
- 保持期間: 7日（`tempo.retention`。トレースは容量に対して価値の減衰が早いため短め）
- OTLP受信(gRPC:4317 / HTTP:4318)はchartデフォルトで有効。Grafana Alloyからのトレース転送先

## デプロイ後の確認

```bash
kubectl -n monitoring get pods -l app.kubernetes.io/name=tempo

# Grafana AlloyからのOTLPトレースが届いているか確認 (GrafanaのTempo Exploreでも確認可能)
kubectl -n monitoring port-forward svc/tempo 3200:3200 &
curl -s localhost:3200/api/search | jq
```
