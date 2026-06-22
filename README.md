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

### シークレットの登録例 (Mastodon)

Mastodon デプロイ前に以下を GCP Secret Manager に登録する。

```bash
# 1. DB パスワード (任意の強パスワード)
gcloud secrets create mastodon-db-password --data-file=- <<< "<your-db-password>"

# 2. SECRET_KEY_BASE
SECRET_KEY_BASE=$(docker run --rm ghcr.io/mastodon/mastodon:v4.5.9 bin/rails secret)
gcloud secrets create mastodon-secret-key-base --data-file=- <<< "$SECRET_KEY_BASE"

# 3. VAPID 鍵ペア
VAPID=$(docker run --rm ghcr.io/mastodon/mastodon:v4.5.9 bin/rails mastodon:webpush:generate_vapid_key)
VAPID_PRIVATE=$(echo "$VAPID" | grep VAPID_PRIVATE_KEY | cut -d= -f2)
VAPID_PUBLIC=$(echo "$VAPID" | grep VAPID_PUBLIC_KEY | cut -d= -f2)
gcloud secrets create mastodon-vapid-private-key --data-file=- <<< "$VAPID_PRIVATE"
gcloud secrets create mastodon-vapid-public-key --data-file=- <<< "$VAPID_PUBLIC"

# 4. ActiveRecord Encryption 鍵
ARE=$(docker run --rm ghcr.io/mastodon/mastodon:v4.5.9 bin/rails db:encryption:init)
ARE_PRIMARY=$(echo "$ARE" | grep primary_key | awk '{print $2}')
ARE_DETER=$(echo "$ARE" | grep deterministic_key | awk '{print $2}')
ARE_SALT=$(echo "$ARE" | grep key_derivation_salt | awk '{print $2}')
gcloud secrets create mastodon-are-primary-key --data-file=- <<< "$ARE_PRIMARY"
gcloud secrets create mastodon-are-deterministic-key --data-file=- <<< "$ARE_DETER"
gcloud secrets create mastodon-are-derivation-salt --data-file=- <<< "$ARE_SALT"

# 5. Cloudflare R2 認証情報 (Cloudflare ダッシュボードで発行した S3 互換 API トークン)
gcloud secrets create mastodon-r2-access-key-id --data-file=- <<< "<r2-access-key-id>"
gcloud secrets create mastodon-r2-secret-access-key --data-file=- <<< "<r2-secret-access-key>"
```

その後、`apps/mastodon/values.yaml` の R2 bucket/endpoint を実際の値に更新し、push する。

### Mastodon 全文検索 (Elasticsearch) を後から有効化する手順

1. `apps/mastodon/resources/elasticsearch.yaml` を追加:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mastodon-es
  namespace: mastodon
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: openebs-hostpath
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mastodon-es
  namespace: mastodon
  labels:
    app: mastodon-es
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mastodon-es
  template:
    metadata:
      labels:
        app: mastodon-es
    spec:
      containers:
        - name: elasticsearch
          image: docker.elastic.co/elasticsearch/elasticsearch:8.18.2
          env:
            - name: discovery.type
              value: single-node
            - name: xpack.security.enabled
              value: "false"
            - name: ES_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
          ports:
            - containerPort: 9200
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
          resources:
            requests:
              memory: 1Gi
            limits:
              memory: 2Gi
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mastodon-es
---
apiVersion: v1
kind: Service
metadata:
  name: mastodon-es
  namespace: mastodon
spec:
  selector:
    app: mastodon-es
  ports:
    - port: 9200
      targetPort: 9200
```

2. `apps/mastodon/values.yaml` の `elasticsearch` セクションを変更:

```yaml
elasticsearch:
  enabled: true
  hostname: mastodon-es
  port: 9200
  tls: false
```

3. mainブランチにpushしてArgoCD syncを待つ。

4. sync完了後、インデックスを作成:

```bash
kubectl exec -n mastodon deploy/mastodon-web -- bin/tootctl search deploy
```
