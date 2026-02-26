# Smart Mobility Dashboard - Kubernetes Manifests

Smart Mobility DashboardアプリケーションをArgoCDでデプロイするためのKubernetesマニフェストリポジトリです。

## ディレクトリ構成

```
smart-mobility-dashboard-manifests/
├── argocd/                    # ArgoCD Application定義
│   ├── application-dev.yaml   # 開発環境用Application
│   └── application-prod.yaml  # 本番環境用Application
├── base/                      # 基本のKubernetesマニフェスト
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   └── frontend-route.yaml
└── overlays/                  # 環境別オーバーレイ
    ├── dev/                   # 開発環境
    │   └── kustomization.yaml
    └── prod/                  # 本番環境
        └── kustomization.yaml
```

## 前提条件

- OpenShiftクラスタ
- ArgoCD がインストール済み
- コンテナイメージがレジストリにプッシュ済み

## 使用方法

### 1. ArgoCDへのApplicationの登録

開発環境:
```bash
kubectl apply -f argocd/application-dev.yaml
```

本番環境:
```bash
kubectl apply -f argocd/application-prod.yaml
```

### 2. 手動でのデプロイ確認（Kustomize）

開発環境のマニフェストをプレビュー:
```bash
kustomize build overlays/dev
```

本番環境のマニフェストをプレビュー:
```bash
kustomize build overlays/prod
```

直接適用する場合:
```bash
kubectl apply -k overlays/dev
```

## 環境別設定

| 設定項目 | 開発環境 (dev) | 本番環境 (prod) |
|---------|---------------|----------------|
| Namespace | smart-mobility-dev | smart-mobility-prod |
| Frontend Replicas | 1 | 2 |
| Backend Replicas | 1 | 2 |
| Image Tag | dev | latest |

## イメージレジストリ

デフォルトでは以下のイメージを使用します:
- Backend: `image-registry.openshift-image-registry.svc:5000/smart-mobility/backend`
- Frontend: `image-registry.openshift-image-registry.svc:5000/smart-mobility/frontend`

本番環境では外部レジストリを使用する場合、overlays/prod/kustomization.yaml を編集してください。
