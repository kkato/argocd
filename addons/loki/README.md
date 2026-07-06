# Loki

ログ集約（SingleBinary/monolithic mode）。オブジェクトストレージは使わず、openebs-hostpath上のfilesystemバックエンドで運用する。

- chart: [`grafana/loki`](https://github.com/grafana/loki) 7.0.0
- ストレージ: `openebs-hostpath`上のfilesystemバックエンド
- 永続化: `openebs-hostpath` 30Gi（実データを恒久保持するため）
- 保持期間: 30日（`limits_config.retention_period` + `compactor.retention_enabled`）
- 前段のnginx gatewayは無効化し、singleBinaryが直接3100番で応答する

## デプロイ後の確認

```bash
kubectl -n monitoring get pods -l app.kubernetes.io/name=loki

# Grafana Alloyから収集されたログがLokiに届いているか確認
kubectl -n monitoring port-forward svc/loki 3100:3100 &
curl -s -G localhost:3100/loki/api/v1/labels | jq
```
