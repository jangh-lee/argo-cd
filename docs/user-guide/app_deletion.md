# 애플리케이션(앱) 삭제
앱은 cascade 옵션을 사용해 지울 수 있습니다. **cascade 삭제**는 app 뿐만 아니라 관련 리소스를 모두 삭제하는 것입니다.

## `argocd` 바이너리를 사용한 삭제


cascade 옵션을 사용하지 않고 삭제:

```bash
argocd app delete APPNAME --cascade=false
```

cascade 옵션을 사용한 삭제:

```bash
argocd app delete APPNAME --cascade
```

또는 다음과 같이 cascade를 적지 않아도 됩니다.

```bash
argocd app delete APPNAME
```
## `kubectl`을 활용한 삭제

cascade 옵션이 없이 삭제하려면, finalizer를 적용하지 않고 삭제합니다:

```bash
kubectl patch app APPNAME  -p '{"metadata": {"finalizers": null}}' --type merge
kubectl delete app APPNAME
```

cascade 옵션을 사용하기 위해서는 finalizer를 적용합니다. 예를들어 `kubectl patch`를 사용해 적용할 수 있습니다: 

```bash
kubectl patch app APPNAME  -p '{"metadata": {"finalizers": ["resources-finalizer.argocd.argoproj.io"]}}' --type merge
kubectl delete app APPNAME
```
## Finalizer 삭제에 관하여

```yaml
metadata:
  finalizers:
    # The default behaviour is foreground cascading deletion
    - resources-finalizer.argocd.argoproj.io
    # Alternatively, you can use background cascading deletion
    # - resources-finalizer.argocd.argoproj.io/background
```

finalizer를 사용해 애플리케이션을 삭제하면, Argo CD 애플리케이션 컨트롤러는 애플리케이션 리소스에 대해 cascade 삭제를 진행합니다.
When deleting an Application with this finalizer, the Argo CD application controller will perform a cascading delete of the Application's resources.

finalizer를 적용하게 되면 [the App of Apps 패턴](../operator-manual/cluster-bootstrapping.md#cascading-deletion) 을 적용할 때 cascade 삭제를 적용할 수 있습니다.
Adding the finalizer enables cascading deletes when implementing [the App of Apps pattern](../operator-manual/cluster-bootstrapping.md#cascading-deletion).


기본적인 cascade 삭제에서 propagation policy 설정은 [foreground cascading deletion](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#foreground-deletion)를 참고하면 됩니다.

Argo CD는 `resources-finalizer.argocd.argoproj.io/background`가 설정되어 있으면 [background cascading deletion](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#background-deletion)을 수행합니다.

만약 `argocd app delete`명령을 `--cascade` 옵션과 함께 실행하면, finalizer가 자동으로 추가됩니다.
propagation 정책은 `--propagation-policy <foreground|background>` 옵션을 사용해서 추가할 수 있습니다.
