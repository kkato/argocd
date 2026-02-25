# k8s

Kubernetesクラスタのマニフェスト管理リポジトリ。ArgoCDによるGitOpsで運用する。

## ディレクトリ構成

```
k8s/
├── apps/          # 自作アプリケーション (素のK8sマニフェスト)
│   └── recipe-api/
├── addons/        # クラスタaddon (addonごとにディレクトリ)
│   ├── argocd/
│   ├── external-secrets/
│   └── cloudflare-tunnel-ingress-controller/
└── argocd/        # ArgoCD Application定義
    ├── root.yaml  # App of Apps ルートApplication
    ├── apps/      # apps/ 配下のアプリ用
    └── addons/    # addons/ 配下のaddon用
```

### apps/

自作アプリケーションのK8sマニフェストを配置する。アプリケーションごとにサブディレクトリを作成する。

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

### 自作アプリの追加

1. `apps/<app-name>/` にK8sマニフェストを配置
2. `argocd/apps/<app-name>.yaml` にArgoCD Application定義を追加
3. mainブランチにpushすると、ArgoCDが自動でsyncする

### 自作アプリの更新

`apps/<app-name>/` 配下のマニフェストを変更してpushするだけで、ArgoCDが自動検知・syncする。

## シークレット管理

External Secrets Operator + Google Cloud Secret Manager でシークレットを管理する。

### 初期設定

1. GCPでService Accountを作成し、Secret Manager Admin ロールを付与
2. Service Accountキー (JSON) をK8s Secretとして登録：

```bash
kubectl create namespace external-secrets
kubectl create secret generic gcp-sa-key -n external-secrets \
  --from-file=credentials.json=<path-to-sa-key.json>
```

3. ArgoCD が ClusterSecretStore と ExternalSecret を自動syncする

### シークレットの登録例 (Cloudflare)

```bash
# GCP Secret Managerにシークレットを作成
gcloud secrets create cloudflare-api-token --data-file=- <<< "<your-api-token>"
gcloud secrets create cloudflare-account-id --data-file=- <<< "<your-account-id>"
gcloud secrets create cloudflare-tunnel-name --data-file=- <<< "cloudflare-ingress"
```
