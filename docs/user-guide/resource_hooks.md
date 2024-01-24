# 리소스 훅(Resource Hooks)
## 개요

Synchronization can be configured using resource hooks. Hooks are ways to run scripts before, during,
and after a Sync operation. Hooks can also be run if a Sync operation fails at any point. Some use cases for hooks are:

* Using a `PreSync` hook to perform a database schema migration before deploying a new version of the app.
* Using a `Sync` hook to orchestrate a complex deployment requiring more sophistication than the
Kubernetes rolling update strategy.
* Using a `PostSync` hook to run integration and health checks after a deployment.
* Using a `SyncFail` hook to run clean-up or finalizer logic if a Sync operation fails. _`SyncFail` hooks are only available starting in v1.2_

## 사용법

리소스 훅은 쿠버네티스 매니페스트를 통해 간단하게 구현할 수 있습니다. Argo CD 애플리케이션의 어노테이션에 `argocd.argoproj.io/hook`를 넣어서 사용하면 됩니다. 에를 들어:
Hooks are simply Kubernetes manifests tracked in the source repository of your Argo CD Application annotated with `argocd.argoproj.io/hook`, e.g.:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: schema-migrate-
  annotations:
    argocd.argoproj.io/hook: PreSync
```

Sync(동기화) 작동 간에, Argo CD는 
During a Sync operation, Argo CD will apply the resource during the appropriate phase of the
deployment. Hooks can be any type of Kubernetes resource kind, but tend to be Pod,
[Job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)
or [Argo Workflows](https://github.com/argoproj/argo). Multiple hooks can be specified as a comma
separated list.

훅의 정의는 다음과 같습니다.

| Hook | Description |
|------|-------------|
| `PreSync` | Executes prior to the application of the manifests. |
| `Sync`  | Executes after all `PreSync` hooks completed and were successful, at the same time as the application of the manifests. |
| `Skip` | Indicates to Argo CD to skip the application of the manifest. |
| `PostSync` | Executes after all `Sync` hooks completed and were successful, a successful application, and all resources in a `Healthy` state. |
| `SyncFail` | Executes when the sync operation fails. _Available starting in v1.2_ |

### 이름 생성

Named hooks (i.e. ones with `/metadata/name`) will only be created once. If you want a hook to be re-created each time either use `BeforeHookCreation` policy (see below) or `/metadata/generateName`. 

## 선택적 동기화
훅은 선택적 동기화 동안에는 실행되지 않습니다.
Hooks are not run during [selective sync](selective_sync.md).

## 훅 삭제 정책
훅은 `argocd.argoproj.io/hook-delete-policy` 어노테이션을 통해 자동으로 삭제할 수 있습니다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: integration-test-
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

The following policies define when the hook will be deleted.

| Policy | Description |
|--------|-------------|
| `HookSucceeded` | The hook resource is deleted after the hook succeeded (e.g. Job/Workflow completed successfully). |
| `HookFailed` | The hook resource is deleted after the hook failed. |
| `BeforeHookCreation` | Any existing hook resource is deleted before the new one is created (since v1.3). It is meant to be used with `/metadata/name`. |

Note that if no deletion policy is specified, ArgoCD will automatically assume `BeforeHookCreation` rules.

### Sync Status with Jobs/Workflows with Time to Live (ttl)

Jobs support the [`ttlSecondsAfterFinished`](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
field in the spec, which let their respective controllers delete the Job after it completes. Argo Workflows support a 
[`ttlStrategy`](https://argoproj.github.io/argo-workflows/fields/#ttlstrategy) property that also allow a Workflow to be 
cleaned up depending on the ttl strategy chosen.

Using either of the properties above can lead to Applications being OutOfSync. This is because Argo CD will detect a difference 
between the Job or Workflow defined in the git repository and what's on the cluster since the ttl properties cause deletion of the resource after completion.

However, using deletion hooks instead of the ttl approaches mentioned above will prevent Applications from having a status of 
OutOfSync even though the Job or Workflow was deleted after completion.

## Using A Hook To Send A Slack Message

The following example uses the Slack API to send a Slack message when sync completes or fails:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: app-slack-notification-
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: slack-notification
        image: curlimages/curl
        command:
          - "curl"
          - "-X"
          - "POST"
          - "--data-urlencode"
          - "payload={\"channel\": \"#somechannel\", \"username\": \"hello\", \"text\": \"App Sync succeeded\", \"icon_emoji\": \":ghost:\"}"
          - "https://hooks.slack.com/services/..."
      restartPolicy: Never
  backoffLimit: 2
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: app-slack-notification-fail-
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: slack-notification
        image: curlimages/curl
        command: 
          - "curl"
          - "-X"
          - "POST"
          - "--data-urlencode"
          - "payload={\"channel\": \"#somechannel\", \"username\": \"hello\", \"text\": \"App Sync failed\", \"icon_emoji\": \":ghost:\"}"
          - "https://hooks.slack.com/services/..."
      restartPolicy: Never
  backoffLimit: 2
```
