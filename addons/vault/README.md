# Vault

HashiCorp Vault（HA Raft, 3 replica）をKubernetes内にセルフホストし、External Secrets Operatorのシークレットストアとして使う。`vault.kkato.app` で Cloudflare Access（Zero Trust）越しに公開する。TLS は Cloudflare エッジで終端。CLI/UI ともに `cloudflared access tcp` 経由で接続する。

## デプロイ後の手動準備

### 初期セットアップ（初回のみ）

ArgoCD が Vault Pod をデプロイした後、手動で初期化・設定を行う。

```bash
# cloudflared でローカルにトンネルを張る（別ターミナルで起動しっぱなしにする）
# 初回はブラウザで Cloudflare Access 認証が必要。以降は Service Token で自動認証。
cloudflared access tcp --hostname vault.kkato.app --url 127.0.0.1:8200 &
export VAULT_ADDR=http://127.0.0.1:8200

# 緊急時フォールバック（k8s ノードに直接アクセスできる環境のみ）:
# kubectl -n vault port-forward svc/vault 8200:8200 &

# 初期化（unsealキー5つ・閾値3）。出力されるキーと root token は安全な場所にオフライン保管すること。
vault operator init -key-shares=5 -key-threshold=3

# unseal（3つのキーを順に入力）
vault operator unseal
vault operator unseal
vault operator unseal

vault login <root-token>

# KV v2 有効化
vault secrets enable -path=secret kv-v2

# Kubernetes auth method（ESO が SA JWT で認証するために必要）
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc"

# ESO 用ポリシー
vault policy write external-secrets - <<'EOF'
path "secret/data/*"     { capabilities = ["read"] }
path "secret/metadata/*" { capabilities = ["read", "list"] }
EOF

# ESO の ServiceAccount にロールを紐付け
vault write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=external-secrets \
  ttl=1h
```

### unseal（Pod 再起動後）

Vault Pod が再起動するたびに手動 unseal が必要。

```bash
# cloudflared でローカルにトンネルを張る（別ターミナルで起動しっぱなしにする）
cloudflared access tcp --hostname vault.kkato.app --url 127.0.0.1:8200 &
export VAULT_ADDR=http://127.0.0.1:8200
vault operator unseal  # ×3（保管しているキーを使用）
```
