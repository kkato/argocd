# MinIO

Loki / Mimir / Tempo が使うS3互換オブジェクトストレージ。`minio` namespaceに単一ノード(standalone mode)で運用する。

- chart: [`minio/minio`](https://charts.min.io/) 5.4.0
- 永続化: `openebs-hostpath` 100Gi
- バケット: `loki` / `mimir` / `tempo`（起動時に自動作成）

## デプロイ前の手動準備

### root クレデンシャルの登録

```bash
vault kv put secret/minio/root \
  root-user=<value> \
  root-password=<value>
```

## デプロイ後の手動準備

### 監視スタック用ユーザーの作成

Mimir / Loki / Tempo が共通で使う最小権限ユーザーを、MinIO起動後に作成する。

```bash
kubectl -n minio port-forward svc/minio 9000:9000 &

# root クレデンシャルは vault kv get secret/minio/root で確認する
mc alias set local http://127.0.0.1:9000 <root-user> <root-password>

# loki/mimir/tempo バケットにのみ read/write可能なユーザーを作成
mc admin user add local <access-key> <secret-key>
mc admin policy create local monitoring-rw - <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": [
        "arn:aws:s3:::loki", "arn:aws:s3:::loki/*",
        "arn:aws:s3:::mimir", "arn:aws:s3:::mimir/*",
        "arn:aws:s3:::tempo", "arn:aws:s3:::tempo/*"
      ]
    }
  ]
}
EOF
mc admin policy attach local monitoring-rw --user <access-key>

# 発行したクレデンシャルを Vault に登録（Mimir/Loki/Tempo が ExternalSecret 経由で参照）
vault kv put secret/minio/monitoring \
  access-key=<access-key> \
  secret-key=<secret-key>
```

## デプロイ後の確認

```bash
kubectl -n minio get pods
mc ls local/
```
