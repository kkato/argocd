# k8s

Kubernetesクラスタのマニフェスト管理リポジトリ。ArgoCDによるGitOpsで運用する。

## ディレクトリ構成

```
k8s/
├── apps/          # 自作アプリケーション (素のK8sマニフェスト)
├── addons/        # クラスタaddon (Helmfile管理)
└── argocd/        # ArgoCD Application定義
    ├── apps/      # apps/ 配下のアプリ用
    └── addons/    # addons/ 配下のaddon用
```

### apps/

自作アプリケーションのK8sマニフェストを配置する。アプリケーションごとにサブディレクトリを作成する。

### addons/

ArgoCD、Ingress Controllerなどのクラスタaddonを管理する。Helmfileで管理し、`helmfile sync` で適用する。

### argocd/

ArgoCD Application CRDを配置する。このリポジトリ内のマニフェストをArgoCD経由でsync対象にするための定義。

## 運用方法

### addonの追加・更新

`addons/helmfile.yaml` を編集し、手動で適用する。

```bash
cd addons
helmfile sync
```

### 自作アプリの追加

1. `apps/<app-name>/` にK8sマニフェストを配置
2. `argocd/apps/<app-name>.yaml` にArgoCD Application定義を追加
3. mainブランチにpushすると、ArgoCDが自動でsyncする

### 自作アプリの更新

`apps/<app-name>/` 配下のマニフェストを変更してpushするだけで、ArgoCDが自動検知・syncする。
