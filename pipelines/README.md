# Tekton Pipelines

Tekton Pipelinesの詳細については、リポジトリルートの [README.md](../README.md) を参照してください。

## クイックリファレンス

```bash
# パイプラインのデプロイ
oc apply -k pipelines/

# パイプラインの実行
oc create -f pipelines/frontend-pipelinerun.yaml
oc create -f pipelines/backend-pipelinerun.yaml

# 実行状況の確認
oc get pipelinerun -n smart-mobility-dev
tkn pipelinerun logs <pipelinerun-name> -n smart-mobility-dev
```
