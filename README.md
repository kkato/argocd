# k8s

Kubernetesクラスタのマニフェスト管理リポジトリ。ArgoCDによるGitOpsで運用する。

## ディレクトリ構成

```
k8s/
├── apps/          # アプリケーション (Helm values + 素のK8sマニフェスト)
│   └── mastodon/
│       ├── values.yaml      # Helm values (ArgoCD ref: values で参照)
│       └── resources/       # 素マニフェスト (CNPG Cluster, Redis, ExternalSecret 等)
├── addons/        # クラスタaddon (addonごとにディレクトリ)
│   ├── argocd/
│   ├── external-secrets/
│   ├── cloudflare-tunnel-ingress-controller/
│   ├── openebs/
│   └── cloudnative-pg/
└── argocd/        # ArgoCD Application定義
    ├── root.yaml  # App of Apps ルートApplication
    ├── apps/      # apps/ 配下のアプリ用
    └── addons/    # addons/ 配下のaddon用
```

### addons/

ArgoCD、External Secrets Operatorなどのクラスタaddonを管理する。各addonはArgoCD Applicationで直接Helm chartを参照する。helmfile.yamlはbootstrap用として残す。

各addonディレクトリには以下を配置する：
- `helmfile.yaml` - Helm chart定義（bootstrap・ローカル確認用）
- `values.yaml` - Helm chartのカスタムvalues
- 追加のK8sマニフェスト（ExternalSecret、ClusterSecretStoreなど）

### argocd/

ArgoCD Application CRDを配置する。このリポジトリ内のマニフェストをArgoCD経由でsync対象にするための定義。

`root.yaml` がApp of AppsパターンのルートApplicationで、`argocd/` 配下のすべてのApplication定義を再帰的に監視・自動syncする。新しいApplication YAMLを追加してpushするだけでArgoCDに自動登録される。

## 運用方法

### 初期セットアップ (bootstrap)

```bash
# 1. ArgoCD をインストール
cd addons/argocd
helmfile sync

# 2. ルートApplicationを適用 (これが最初で最後の手動apply)
kubectl apply -f argocd/root.yaml
```

### addonの追加

1. `addons/<addon-name>/` ディレクトリを作成
2. `addons/<addon-name>/helmfile.yaml` にHelm chart定義を配置
3. `addons/<addon-name>/values.yaml` にHelmのカスタムvaluesを配置
4. `argocd/addons/<addon-name>.yaml` にArgoCD Application定義を追加
5. mainブランチにpushすると、ArgoCDが自動でsyncする

### アプリの追加 (Helm chart + 周辺リソース)

Mastodonのように「Helm chart + CNPG/Redis等の周辺リソース」をまとめてデプロイするパターン:

1. `apps/<app-name>/values.yaml` にHelm values を配置
2. `apps/<app-name>/resources/` に周辺リソースの素マニフェストを配置
3. `argocd/apps/<app-name>.yaml` — Helm chart の multi-source Application を追加
4. `argocd/apps/<app-name>-resources.yaml` — 素マニフェスト用 directory Application を追加
5. mainブランチにpushすると、ArgoCDが自動でsyncする

## リポジトリ外で管理されるコンポーネント

以下のコンポーネントはこのリポジトリでは管理していない。

| コンポーネント | namespace | 管理方法 | 備考 |
|---|---|---|---|
| Flannel (CNI) | `kube-flannel` | 手動 (`kubectl apply`) | CNIはクラスタのライフライン。自動syncで壊れるとノード間通信が全断するため手動管理が安全 |
| CoreDNS | `kube-system` | kubeadm | kubeadmが管理するコントロールプレーンコンポーネント |
| kube-proxy | `kube-system` | kubeadm | kubeadmが管理するコントロールプレーンコンポーネント |

## シークレット管理

External Secrets Operator + HashiCorp Vault でシークレットを管理する。
VaultはKubernetes内にセルフホスト（`vault` namespace）し、クラスタ外には公開しない。

### Vault の初期セットアップ（初回のみ）

ArgoCD が Vault Pod をデプロイした後、手動で初期化・設定を行う。

```bash
# port-forward でローカルから接続
kubectl -n vault port-forward svc/vault 8200:8200 &
export VAULT_ADDR=http://127.0.0.1:8200

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

### Vault の unseal（Pod 再起動後）

Vault Pod が再起動するたびに手動 unseal が必要。

```bash
kubectl -n vault port-forward svc/vault 8200:8200 &
export VAULT_ADDR=http://127.0.0.1:8200
vault operator unseal  # ×3（保管しているキーを使用）
```

### シークレットの登録例 (Cloudflare)

```bash
export VAULT_ADDR=http://127.0.0.1:8200
vault login <root-token>

vault kv put secret/cloudflare \
  api_token="<your-api-token>" \
  account_id="<your-account-id>" \
  tunnel_name="cloudflare-ingress"
```

各アプリのシークレット登録手順はそれぞれのディレクトリの README を参照:
- [apps/mastodon/README.md](apps/mastodon/README.md)
