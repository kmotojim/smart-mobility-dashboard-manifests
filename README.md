# Smart Mobility Dashboard - Kubernetes Manifests

Smart Mobility DashboardアプリケーションをOpenShift上にデプロイするためのKubernetesマニフェストリポジトリです。
ArgoCDによるGitOpsデプロイとTekton Pipelinesによるコンテナイメージビルドをサポートしています。

## ディレクトリ構成

```
smart-mobility-dashboard-manifests/
├── argocd/                    # ArgoCD Application定義
│   ├── appproject.yaml        # AppProject定義
│   ├── application-dev.yaml   # 開発環境用Application
│   └── application-prod.yaml  # 本番環境用Application
├── base/                      # 基本のKubernetesマニフェスト
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── frontend-route.yaml
│   └── e2e-test-job.yaml      # E2Eテスト（ArgoCD Post-Sync Hook）
├── overlays/                  # 環境別オーバーレイ
│   ├── dev/                   # 開発環境
│   │   └── kustomization.yaml
│   └── prod/                  # 本番環境
│       └── kustomization.yaml
└── pipelines/                 # Tekton Pipelines
    ├── pvc.yaml               # パイプライン用PVC
    ├── *-pipeline.yaml        # パイプライン定義
    └── *-pipelinerun.yaml     # パイプライン実行定義
```

## 前提条件

### 必須
- OpenShiftクラスタ
- ArgoCD がインストール済み
- OpenShift Pipelines Operator がインストール済み

### 確認コマンド
```bash
# OpenShift Pipelines Operatorの確認
oc get csv -n openshift-operators | grep pipelines

# ArgoCDの確認
oc get pods -n openshift-gitops
```

---

## ArgoCD セットアップ

### 1. ArgoCDへのApplicationの登録

開発環境:
```bash
oc apply -f argocd/application-dev.yaml
```

本番環境:
```bash
oc apply -f argocd/application-prod.yaml
```

### 2. 手動でのデプロイ確認（Kustomize）

マニフェストをプレビュー:
```bash
# 開発環境
kustomize build overlays/dev

# 本番環境
kustomize build overlays/prod
```

直接適用する場合:
```bash
oc apply -k overlays/dev
```

### 環境別設定

| 設定項目 | 開発環境 (dev) | 本番環境 (prod) |
|---------|---------------|----------------|
| Namespace | smart-mobility-dev | smart-mobility-prod |
| Frontend Replicas | 1 | 2 |
| Backend Replicas | 1 | 2 |
| Image Tag | dev | latest |

---

## Tekton Pipelines

smart-mobility-dev環境でコンテナイメージをビルドし、外部レジストリにpushするためのTekton Pipelinesです。

### パイプライン構成

#### Image Build Pipelines（イメージビルドのみ）
- `frontend-image-build-pipeline`: フロントエンドイメージのビルド・プッシュ
- `backend-image-build-pipeline`: バックエンドイメージのビルド・プッシュ

**タスクフロー:**
1. **git-clone**: GitHubリポジトリからソースコードをクローン
2. **buildah**: Dockerfileを使用してイメージをビルドし、OpenShift内部レジストリにプッシュ
3. **skopeo-copy**: 内部レジストリから外部レジストリ（Quay.io）にイメージをコピー

#### SonarQube Pipelines（静的解析 + イメージビルド）
- `frontend-sonarqube-pipeline`: フロントエンドのSonarQube静的解析とイメージビルド・プッシュ
- `backend-sonarqube-pipeline`: バックエンドのSonarQube静的解析とイメージビルド・プッシュ

**タスクフロー:**
1. **git-clone**: GitHubリポジトリからソースコードをクローン
2. **sonarqube-scan**: SonarQubeによる静的コード解析
3. **buildah**: Dockerfileを使用してイメージをビルドし、OpenShift内部レジストリにプッシュ
4. **skopeo-copy**: 内部レジストリから外部レジストリ（Quay.io）にイメージをコピー

#### CI Pipeline（単体テスト + 静的解析 + イメージビルド）
- `backend-ci-pipeline`: バックエンドの単体テスト、SonarQube静的解析、イメージビルド・プッシュ

**タスクフロー:**
1. **git-clone**: GitHubリポジトリからソースコードをクローン
2. **unit-test**: Maven による単体テスト実行（`mvn test`）
3. **sonarqube-scan**: SonarQubeによる静的コード解析（テスト結果も含む）
4. **buildah**: Dockerfileを使用してイメージをビルドし、OpenShift内部レジストリにプッシュ
5. **skopeo-copy**: 内部レジストリから外部レジストリ（Quay.io）にイメージをコピー

### 外部レジストリ認証の設定

Quay.ioなどの外部レジストリにpushするには、認証情報が必要です。

```bash
# Quay.ioのロボットアカウントまたは認証トークンでSecretを作成
oc create secret docker-registry quay-credentials \
  --docker-server=quay.io \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n smart-mobility-dev

# pipeline ServiceAccountにSecretをリンク
oc secrets link pipeline quay-credentials --for=pull,mount -n smart-mobility-dev
```

### パイプラインのデプロイ

```bash
# pipelinesディレクトリのリソースを適用
oc apply -k pipelines/
```

または個別に適用:
```bash
oc apply -f pipelines/pvc.yaml
oc apply -f pipelines/frontend-image-build-pipeline.yaml
oc apply -f pipelines/backend-image-build-pipeline.yaml
```

### パイプラインの実行

#### Image Build Pipeline

```bash
# CLI経由での実行
oc create -f pipelines/frontend-pipelinerun.yaml
oc create -f pipelines/backend-pipelinerun.yaml

# tkn CLI経由での実行
tkn pipeline start frontend-image-build-pipeline \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev

# 特定のタグでビルド
tkn pipeline start frontend-image-build-pipeline \
  -p IMAGE_TAG=v1.0.0 \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev
```

#### SonarQube Pipeline

```bash
# CLI経由での実行（PipelineRunファイルのSOAR_TOKENを事前に更新）
oc create -f pipelines/frontend-sonarqube-pipelinerun.yaml
oc create -f pipelines/backend-sonarqube-pipelinerun.yaml

# tkn CLI経由での実行
tkn pipeline start frontend-sonarqube-pipeline \
  -p SONAR_PROJECT_KEY=smart-mobility-frontend \
  -p SONAR_TOKEN=<your-sonar-token> \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev
```

#### CI Pipeline

```bash
# CLI経由での実行
oc create -f pipelines/backend-ci-pipelinerun.yaml

# tkn CLI経由での実行
tkn pipeline start backend-ci-pipeline \
  -p SONAR_PROJECT_KEY=smart-mobility-backend \
  -p SONAR_TOKEN=<your-sonar-token> \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev
```

### パイプライン実行の確認

```bash
# PipelineRunの一覧
oc get pipelinerun -n smart-mobility-dev

# 特定のPipelineRunのログ
tkn pipelinerun logs <pipelinerun-name> -n smart-mobility-dev

# PipelineRunの状態確認
tkn pipelinerun describe <pipelinerun-name> -n smart-mobility-dev
```

### パラメータ一覧

#### 共通パラメータ

| パラメータ | 説明 | デフォルト値 |
|-----------|------|-------------|
| GIT_URL | GitリポジトリURL | https://github.com/kmotojim/smart-mobility-dashboard.git |
| GIT_REVISION | Gitブランチ/タグ/コミット | main |
| DOCKERFILE_PATH | Dockerfileのパス | ./frontend/Dockerfile または ./backend/Dockerfile |
| SOURCE_PATH | ビルドコンテキストのパス | frontend または backend |
| INTERNAL_REGISTRY_URL | OpenShift内部レジストリURL | image-registry.openshift-image-registry.svc:5000/smart-mobility-dev/smart-mobility-* |
| EXTERNAL_REGISTRY_URL | 外部レジストリURL | quay.io/rh_ee_kmotojim/smart-mobility-* |
| IMAGE_TAG | イメージタグ | latest |

#### SonarQube Pipeline 追加パラメータ

| パラメータ | 説明 | デフォルト値 |
|-----------|------|-------------|
| SONAR_HOST_URL | SonarQubeサーバーのURL | http://docker-sonarqube-git.sonarqube.svc.cluster.local:9000 |
| SONAR_PROJECT_KEY | SonarQubeのプロジェクトキー | （必須） |
| SONAR_TOKEN | SonarQubeの認証トークン | （必須） |

---

## E2Eテスト（ArgoCD Post-Sync Hook）

ArgoCDがリソースをSyncするたびに、Post-Sync HookとしてE2Eテストが自動実行されます。

### 動作フロー

```
ArgoCD Sync完了
    ↓
Post-Sync Hook起動
    ↓
[init] wait-for-app: サービス起動待機
    ↓
[main] e2e-test: Cucumber + Playwright実行
    ↓
テスト結果（成功/失敗）
```

### 技術スタック
- **Cucumber**: Gherkin記法（日本語）でテストシナリオを記述
- **Playwright**: ヘッドレスブラウザによるGUI操作
- **JUnit 5**: テスト実行フレームワーク

### E2Eテスト結果の確認

```bash
# E2EテストJobの一覧
oc get jobs -n smart-mobility-dev -l app.kubernetes.io/name=e2e-test

# E2EテストPodのログ確認
oc logs -n smart-mobility-dev -l app.kubernetes.io/name=e2e-test --tail=100

# 最新のE2EテストPodのログを全て表示
oc logs -n smart-mobility-dev $(oc get pods -n smart-mobility-dev -l app.kubernetes.io/name=e2e-test --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
```

### E2Eテストの手動再実行

```bash
# 既存のJobを削除して再作成（ArgoCD経由）
oc delete job e2e-test-job -n smart-mobility-dev
argocd app sync smart-mobility-dashboard-dev

# または直接Jobを実行
oc create -f base/e2e-test-job.yaml -n smart-mobility-dev
```

### テストシナリオ

E2Eテストは `e2e-tests/src/test/resources/features/dashboard.feature` で定義されています。

主なテストカテゴリ：
- **@speedometer**: スピードメーター関連（加速/減速/最大速度）
- **@gear**: ギアインジケーター関連（ギア変更/表示）
- **@warning**: 警告灯関連（速度警告/エンジン警告/シートベルト）
- **@interlock**: 安全インターロック（走行中のギア変更禁止など）
- **@control**: 制御パネル関連（リセット機能）

---

## リセット手順

デモ環境を初期状態に戻すための手順です。

### クイックリセット（アプリケーションのみ）

アプリケーションを再デプロイしてリセットします。

```bash
# ArgoCDアプリケーションを再同期
argocd app sync smart-mobility-dashboard-dev --force

# または、Deploymentを再起動
oc rollout restart deployment/frontend -n smart-mobility-dev
oc rollout restart deployment/backend -n smart-mobility-dev
```

### PipelineRunの削除

過去のパイプライン実行履歴を削除します。

```bash
# 全てのPipelineRunを削除
oc delete pipelinerun --all -n smart-mobility-dev

# 特定のパイプラインのPipelineRunのみ削除
oc delete pipelinerun -l tekton.dev/pipeline=frontend-image-build-pipeline -n smart-mobility-dev
oc delete pipelinerun -l tekton.dev/pipeline=backend-image-build-pipeline -n smart-mobility-dev
```

### E2Eテスト結果の削除

```bash
# E2EテストJobを削除
oc delete job -l app.kubernetes.io/name=e2e-test -n smart-mobility-dev
```

### フルリセット（Namespace削除）

環境を完全にリセットする場合は、Namespaceごと削除して再作成します。

```bash
# 1. ArgoCDアプリケーションを削除
oc delete -f argocd/application-dev.yaml

# 2. Namespaceの削除を確認（ArgoCDが自動削除する場合もあります）
oc delete namespace smart-mobility-dev

# 3. Namespaceが削除されるまで待機
oc wait --for=delete namespace/smart-mobility-dev --timeout=120s

# 4. パイプラインを再デプロイ（Namespaceも作成されます）
oc apply -k pipelines/

# 5. 外部レジストリ認証を再設定
oc create secret docker-registry quay-credentials \
  --docker-server=quay.io \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n smart-mobility-dev
oc secrets link pipeline quay-credentials --for=pull,mount -n smart-mobility-dev

# 6. ArgoCDアプリケーションを再登録
oc apply -f argocd/application-dev.yaml
```

### 本番環境のリセット

```bash
# 1. ArgoCDアプリケーションを削除
oc delete -f argocd/application-prod.yaml

# 2. Namespaceの削除を確認
oc delete namespace smart-mobility-prod

# 3. Namespaceが削除されるまで待機
oc wait --for=delete namespace/smart-mobility-prod --timeout=120s

# 4. ArgoCDアプリケーションを再登録
oc apply -f argocd/application-prod.yaml
```

### 全環境の一括リセット

```bash
# 全てのArgoCDアプリケーションを削除
oc delete -f argocd/

# 両方のNamespaceを削除
oc delete namespace smart-mobility-dev smart-mobility-prod

# 削除完了を待機
oc wait --for=delete namespace/smart-mobility-dev namespace/smart-mobility-prod --timeout=180s

# パイプラインとアプリケーションを再デプロイ
oc apply -k pipelines/
oc apply -f argocd/
```

---

## イメージレジストリ

デフォルトでは以下のイメージを使用します:
- Backend: `image-registry.openshift-image-registry.svc:5000/smart-mobility-dev/smart-mobility-backend`
- Frontend: `image-registry.openshift-image-registry.svc:5000/smart-mobility-dev/smart-mobility-frontend`

外部レジストリ（Quay.io）を使用する場合:
- Backend: `quay.io/rh_ee_kmotojim/smart-mobility-backend`
- Frontend: `quay.io/rh_ee_kmotojim/smart-mobility-frontend`

本番環境で外部レジストリを使用する場合、`overlays/prod/kustomization.yaml` を編集してください。
