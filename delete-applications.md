https://argo-cd.readthedocs.io/en/stable/user-guide/app_deletion/

To cascade delete application resources (delete application but keep their sub resources):
For the technical amongst you, the Argo CD application controller watches for this finalizer:

```
metadata:
  finalizers:
    - resources-finalizer.argocd.argoproj.io
```

=> Nếu thêm finalizers này vào application objects thì sau khi xóa application object => argocd controller sẽ xóa cả các sub resources của application đó.

Argo CD's app controller watches for this and will then delete both the app and its resources.

When you invoke argocd app delete with --cascade, the finalizer is added automatically.

with kubectl:
```
kubectl patch app APPNAME  -p '{"metadata": {"finalizers": ["resources-finalizer.argocd.argoproj.io"]}}' --type merge
kubectl delete app APPNAME
```

