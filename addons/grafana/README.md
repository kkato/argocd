# Grafana

`grafana.kkato.app`（Cloudflare Tunnel経由、TLSはCloudflareエッジで終端）で公開する可視化基盤。

- chart: [`grafana/grafana`](https://github.com/grafana-community/helm-charts) 12.7.2
- データソース: Mimir(Prometheus互換API) / Loki / Tempo を自動プロビジョニング（Tempo→Loki/Mimirの相関設定込み）
- 永続化: `openebs-hostpath` 2Gi（手動作成ダッシュボード保持用）

## デプロイ前の手動準備

### 管理者クレデンシャルの登録

```bash
vault kv put secret/grafana/admin \
  admin-user=admin \
  admin-password=<value>
```

## デプロイ後の確認

`https://grafana.kkato.app` にアクセスし、上記で登録した管理者クレデンシャルでログインする。

Data sources設定画面で Mimir / Loki / Tempo それぞれ「Data source is working」と表示されることを確認する。
