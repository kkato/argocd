# Cloudflare Tunnel Ingress Controller

`kkato.app` の Cloudflare Tunnel経由でクラスタ内サービスを公開するIngressController。`ingressClassName: cloudflare-tunnel` を指定したIngressがルーティング対象になる。TLSはCloudflareエッジで終端する。

## デプロイ前の手動準備

### シークレットの登録例

```bash
# vault-tunnel エイリアス（dotfiles）または cloudflared access tcp でトンネルを起動済みの状態で実行
# export VAULT_ADDR は dotfiles の .zshrc で設定済み（http://127.0.0.1:8200）
vault login <root-token>

vault kv put secret/cloudflare \
  api_token="<your-api-token>" \
  account_id="<your-account-id>" \
  tunnel_name="cloudflare-ingress"
```
