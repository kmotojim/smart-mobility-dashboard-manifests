# Smart Mobility Dashboard - Tekton Pipelines

smart-mobility-dev環境でコンテナイメージをビルドし、外部レジストリにpushするためのTekton Pipelinesです。

## パイプライン構成

### Image Build Pipelines（イメージビルドのみ）
- `frontend-image-build-pipeline`: フロントエンドイメージのビルド・プッシュ
- `backend-image-build-pipeline`: バックエンドイメージのビルド・プッシュ

#### タスクフロー
1. **git-clone**: GitHubリポジトリからソースコードをクローン
2. **buildah**: Dockerfileを使用してイメージをビルドし、OpenShift内部レジストリにプッシュ
3. **skopeo-copy**: 内部レジストリから外部レジストリ（Quay.io）にイメージをコピー

### SonarQube Pipelines（静的解析 + イメージビルド）
- `frontend-sonarqube-pipeline`: フロントエンドのSonarQube静的解析とイメージビルド・プッシュ
- `backend-sonarqube-pipeline`: バックエンドのSonarQube静的解析とイメージビルド・プッシュ

#### タスクフロー
1. **git-clone**: GitHubリポジトリからソースコードをクローン
2. **sonarqube-scan**: SonarQubeによる静的コード解析
   - sonar-project.propertiesの作成
   - Maven compile（Java用）
   - SonarQubeスキャン実行
3. **buildah**: Dockerfileを使用してイメージをビルドし、OpenShift内部レジストリにプッシュ
4. **skopeo-copy**: 内部レジストリから外部レジストリ（Quay.io）にイメージをコピー

### CI Pipeline（単体テスト + 静的解析 + イメージビルド）
- `backend-ci-pipeline`: バックエンドの単体テスト、SonarQube静的解析、イメージビルド・プッシュ

#### タスクフロー
1. **git-clone**: GitHubリポジトリからソースコードをクローン
2. **unit-test**: Maven による単体テスト実行（`mvn test`）
3. **sonarqube-scan**: SonarQubeによる静的コード解析（テスト結果も含む）
4. **buildah**: Dockerfileを使用してイメージをビルドし、OpenShift内部レジストリにプッシュ
5. **skopeo-copy**: 内部レジストリから外部レジストリ（Quay.io）にイメージをコピー

### E2E Test（ArgoCD Post-Sync Hook）
- `e2e-test-job`: ArgoCDのデプロイ完了後に自動実行されるE2Eテスト

#### 実行フロー
ArgoCDがリソースをSyncした後、Post-Sync Hookとして自動実行されます。

1. **wait-for-app** (init container): frontend/backendサービスの起動完了を待機
2. **e2e-test**: Playwright + CucumberによるE2Eテスト実行

#### 技術スタック
- **Cucumber**: Gherkin記法（日本語）でテストシナリオを記述
- **Playwright**: ヘッドレスブラウザによるGUI操作
- **JUnit 5**: テスト実行フレームワーク

## 前提条件

### 1. OpenShift Pipelinesのインストール
```bash
# OpenShift Pipelines Operatorがインストールされていることを確認
oc get csv -n openshift-operators | grep pipelines
```

### 2. 外部レジストリ認証の設定
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

## デプロイ

### パイプラインの適用
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

## パイプラインの実行

### Image Build Pipeline の実行

#### CLI経由での実行
```bash
# フロントエンドパイプラインの実行
oc create -f pipelines/frontend-pipelinerun.yaml

# バックエンドパイプラインの実行
oc create -f pipelines/backend-pipelinerun.yaml
```

#### tkn CLI経由での実行
```bash
# フロントエンドパイプラインの実行
tkn pipeline start frontend-image-build-pipeline \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev

# バックエンドパイプラインの実行
tkn pipeline start backend-image-build-pipeline \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev
```

#### 特定のタグでビルド
```bash
tkn pipeline start frontend-image-build-pipeline \
  -p IMAGE_TAG=v1.0.0 \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev
```

### SonarQube Pipeline の実行

#### CLI経由での実行
```bash
# PipelineRunファイルのSOAR_TOKENを事前に更新してから実行
# フロントエンド
oc create -f pipelines/frontend-sonarqube-pipelinerun.yaml

# バックエンド
oc create -f pipelines/backend-sonarqube-pipelinerun.yaml
```

#### tkn CLI経由での実行
```bash
# フロントエンドSonarQubeパイプラインの実行
tkn pipeline start frontend-sonarqube-pipeline \
  -p SONAR_PROJECT_KEY=smart-mobility-frontend \
  -p SONAR_TOKEN=<your-sonar-token> \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev

# バックエンドSonarQubeパイプラインの実行
tkn pipeline start backend-sonarqube-pipeline \
  -p SONAR_PROJECT_KEY=smart-mobility-backend \
  -p SONAR_TOKEN=<your-sonar-token> \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev
```

### CI Pipeline の実行（単体テスト + 静的解析 + イメージビルド）

#### CLI経由での実行
```bash
# PipelineRunファイルのSOAR_TOKENを事前に更新してから実行
oc create -f pipelines/backend-ci-pipelinerun.yaml
```

#### tkn CLI経由での実行
```bash
# バックエンドCIパイプラインの実行
tkn pipeline start backend-ci-pipeline \
  -p SONAR_PROJECT_KEY=smart-mobility-backend \
  -p SONAR_TOKEN=<your-sonar-token> \
  -w name=workspace,claimName=pipeline-workspace-pvc \
  -n smart-mobility-dev
```

## パイプライン実行の確認
```bash
# PipelineRunの一覧
oc get pipelinerun -n smart-mobility-dev

# 特定のPipelineRunのログ
tkn pipelinerun logs <pipelinerun-name> -n smart-mobility-dev

# PipelineRunの状態確認
tkn pipelinerun describe <pipelinerun-name> -n smart-mobility-dev
```

## パラメータ

### 共通パラメータ

| パラメータ | 説明 | デフォルト値 |
|-----------|------|-------------|
| GIT_URL | GitリポジトリURL | https://github.com/kmotojim/smart-mobility-dashboard.git |
| GIT_REVISION | Gitブランチ/タグ/コミット | main |
| DOCKERFILE_PATH | Dockerfileのパス | ./frontend/Dockerfile または ./backend/Dockerfile |
| SOURCE_PATH | ビルドコンテキストのパス | frontend または backend |
| INTERNAL_REGISTRY_URL | OpenShift内部レジストリURL | image-registry.openshift-image-registry.svc:5000/smart-mobility-dev/smart-mobility-* |
| EXTERNAL_REGISTRY_URL | 外部レジストリURL | quay.io/rh_ee_kmotojim/smart-mobility-* |
| IMAGE_TAG | イメージタグ | latest |

### SonarQube Pipeline 追加パラメータ

| パラメータ | 説明 | デフォルト値 |
|-----------|------|-------------|
| SONAR_HOST_URL | SonarQubeサーバーのURL | http://docker-sonarqube-git.sonarqube.svc.cluster.local:9000 |
| SONAR_PROJECT_KEY | SonarQubeのプロジェクトキー | （必須） |
| SONAR_TOKEN | SonarQubeの認証トークン | （必須） |

## E2Eテスト（ArgoCD Post-Sync Hook）

### 概要
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
