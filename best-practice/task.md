## account webui
username: admin
password: truemoney

Secret object **argocd-secret** sẽ chứa các thông tin username, password này
## Setup repository
1. Through k8s object
- Create secret:
VD:
```
kubectl get secret repo-argo-test-855366235 -oyaml
apiVersion: v1
data:
  password: MzA4MjEwMTk5NA==
  username: bmd1eWVuZ3Q=
kind: Secret
```
Trong đó username và password là của account bitbucket có quyền pull, push trên bitbucket repo đó.

- Update configmap **argocd-cm**

```
kubectl get cm argocd-cm -oyaml
apiVersion: v1
data:
  repositories: |
    - passwordSecret:
        key: password
        name: repo-argo-test-855366235    # Ứng với tên secret object ở trên chứa thông tin này
      url: https://nguyengt@bitbucket.org/nguyengt/argo-test.git
      usernameSecret:
        key: username
        name: repo-argo-test-855366235    # Ứng với tên secret object ở trên chứa thông tin này
```

## Setup application object
- Để tạo application object dễ nhất thì trước tiên ta nên vào giao diện UI web và tiến hành new application
- Khi đó ở namespace ta cài argocd ta thực hiện ```kubectl get application```
  Ta sẽ thấy application object ta vừa tạo, ta sẽ tiến hành sửa object này theo ý ta mong muốn, 1 ví dụ là file app-template.yml ở ngoài

Nếu ta dùng lệnh **argocd app set consumer-app-payment-service --values valuesStg.yaml**
mà gặp lỗi:
```
FATA[0003] rpc error: code = InvalidArgument desc = application spec is invalid: InvalidSpecError: Unable to determine app source type: multiple application sources defined: Helm,Directory
```

Thì ta nên kiểm tra application object và sửa lại.

Nếu trên giao diện web ta tiến hành sync mà không thấy gì xảy ra thì ta nên **kubectl get application**

## Chú ý:
Về mặt bản chất thì argocd sẽ luôn so sánh actual state ở cluster với trạng thái desired state theo git (helm) và đưa ra thay đổi ( tương tự kubectl apply) để match 2 state này với nhau
Tuy nhiên giả sử như ta cập nhật image version hay config nào đó trong deployment trên git mà khi argocd này apply vào cluster => mà khi deploy xong dẫn đến app failed VD như 2 k pull được image, version này bị lỗi dẫn đến helth check failed thì khi đó trạng thái của application object trong argocd sẽ là outofsync và argocd k tự động rollback khi thay đổi này gây chết pod...


Disable prune resource:
Disable pruning for namespaces using at least Prune=false annotation
ref:
https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#no-prune-resources


## Preserve resources

All Application resources created by the ApplicationSet controller (from an ApplicationSet) will contain:
- A .metadata.ownerReferences reference back to the parent ApplicationSet resource
- An Argo CD resources-finalizer.argocd.argoproj.io finalizer in .metadata.finalizers of the Application if .syncPolicy.preserveResourcesOnDeletion is set to false.

The end result is that when an ApplicationSet is deleted, the following occurs (in rough order):
- The ApplicationSet resource itself is deleted
- Any Application resources that were created from this ApplicationSet (as identified by owner reference)
- Any deployed resources (Deployments, Services, ConfigMaps, etc) on the managed cluster, that were created from that Application resource (by Argo CD), will be deleted.
  - Argo CD is responsible for handling this deletion, via the deletion finalizer.
  - To preserve deployed resources, set .syncPolicy.preserveResourcesOnDeletion to true in the ApplicationSet.

Thus the lifecycle of the ApplicationSet, the Application, and the Application's resources, are equivalent.

=> Even if using a non-cascaded delete, the resources-finalizer.argocd.argoproj.io is still specified on the Application. Thus, when the Application is deleted, all of its deployed resources will also be deleted. (The lifecycle of the Application, and its child objects, are still equivalent.)
To prevent the deletion of the resources of the Application, such as Services, Deployments, etc, set .syncPolicy.preserveResourcesOnDeletion to true in the ApplicationSet. This syncPolicy parameter prevents the finalizer from being added to the Application.

ref:
https://argocd-applicationset.readthedocs.io/en/stable/Application-Deletion/#application-pruning-resource-deletion


## Note with PV resources

Do not store multiple objects in a single YAML file, it’s very important! (to avoid mistakes like one document overwrites another when you miss the separator)
Disable pruning via annotation as well as in kind: Application automated synchronization options (prune: false) if using automated
Disable resources overlapping (FailOnSharedResource=true)
Disable kind: Application deletion in ApplicationSet if you decide to use hierarchical structure and ApplicationSet