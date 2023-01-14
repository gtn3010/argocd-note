#!/usr/bin/env sh

set -e

exec java $JAVA_OPTS -cp . org.springframework.boot.loader.JarLauncher
## Notable

New Configmap argocd-cmd-params-cm
Simplify parametrization of Argo CD server processes: an additional optional ConfigMap argocd-cmd-params-cm has been introduced. The ConfigMap pulls together various settings that control the behavior of different Argo CD components.
https://blog.argoproj.io/argo-cd-v2-1-first-release-candidate-is-ready-c1aab7795638

Chú ý: Nếu ta set ApplyOutOfSyncOnly=true cho application thì nếu ở application đó có pod not ready (readiness failed) - không phải crashloopbackoff.
Khi đó nếu ta push lên git, và dù có webhook trigger về argocd thì lúc đó argocd cũng k sync ngay mà sẽ đợi chu kì auto sync mỗi 3 phút.
=> Nên với trường hợp pod not ready , ta có thể cân nhắc bỏ set ApplyOutOfSyncOnly=true cho application.


Khi ta sử dụng prometheus-operator, khi apply crd prometheus sẽ bị lỗi:
```
The CustomResourceDefinition "prometheuses.monitoring.coreos.com" is invalid: metadata.annotations: Too long: must have at most 262144 bytes
```

Follow link này (https://github.com/prometheus-community/helm-charts/issues/1500), ta sẽ sửa bằng cách thêm annotation cho crd để argocd sync:
```
argocd.argoproj.io/sync-options: Replace=true
```

Reason:
- Can you try with kubectl create. The issue is due to large annotation size which will be issue for kubectl apply since it stores last applied config.
- kubectl apply will store kubernetes.io/last-applied-configuration annotation even if we are creating a new resource 

ref:
https://issueexplorer.com/issue/prometheus-operator/prometheus-operator/4355


## Overview
doc: https://argoproj.github.io/argo-cd/operator-manual/high_availability/

The argocd-repo-server is responsible for cloning Git repository, keeping it up to date and generating manifests using the appropriate tool.
The argocd-application-controller uses argocd-repo-server to get generated manifests and Kubernetes API server to get actual cluster state.

argoCD có 3 thành phần chính:
- argocd-server: api server - The API server is a gRPC/REST server which exposes the API consumed by the Web UI, CLI, and CI/CD systems.
- argocd-repo-server: Có sử dụng component redis, lưu repo pull về ở /tmp => clone Git repository, generate Kubernetes YAML and cache. => Chạy các lệnh như helm init --client-only, helm template
  ArgoCD Repository Server is an internal service which maintains a local cache of the Git repository holding the application manifests, and is responsible for generating and returning the Kubernetes manifests. 
  The argocd-repo-server is responsible for cloning Git repository, keeping it up to date and generating manifests using the appropriate tool.
- argocd-application-controller: Chịu trách nhiệm chính đảm bảo cluster mà ta muốn ở desired state theo git => Chạy lệnh kubectl apply... , check thư mục home pod (user argocd) này sẽ có thư mục .helm, .kube
  ArgoCD application controller is a Kubernetes controller that continuously monitors running applications and compares the current, live state against the desired target state (as specified in the repo).

2 thành phần bổ trợ là:
-argocd-redis
-argocd-dex-server

https://medium.com/riskified-technology/gitops-deployment-and-kubernetes-f1ab289efa4b
**ArgoCD in practice**
ArgoCD watches the Git repo and runs the Helm template; these templated files are then compared to the desired state in the cluster — this is the sync status in the application view. If there are differences, ArgoCD can apply the templated files and change the K8s desired state — this can be manual or automatic. ArgoCD also watches the live K8s objects and compares them to the K8s desired state — this is the health status in the application view.
**No `$helm install`**
ArgoCD doesn’t use `$helm install`. It is not a Helm wrapper, but rather a declarative GitOps deploy tool. The way you construct the templated K8s objects in Git doesn’t need to be related to the way you deploy them to the cluster. We use Helm for templating, and we use ArgoCD for deploying and managing the cluster. Each tool is the best for its purpose. `$kubectl apply` is also the easiest way to change the desired state in the cluster. Using it helps us reduce the magic around the deployment process.

**Tóm lại**
Bản chất argoCD hỗ trợ các tool như helm, ksonnet,kustomize là do argoCD chỉ coi các tool này như công cụ để templating , bản chất việc deploy sẽ là chạy kubectl apply
VD:
- Với helm, argoCD sẽ chạy helm template => generate ra yaml => kubectl validate sau đó kubectl apply ( ArgoCD uses helm template to generate the manifests for deploying charts to the cluster. It does not require nor use the server-side tiller component. - ref: https://github.com/argoproj/argo-cd/issues/2383)
- Vì vậy ta có kubectl get cm hay kubectl get secret sẽ k thấy config map hay secret nào liên quan đến helm release deploy
chú ý: ArgoCD does not use Helm release. It only uses Helm templates or similar, so you will not be able to use Helm cli to manage your resources anymore.

Application object: Đại diện cho 1 application: sẽ có các thông tin như dest k8s cluster để deploy, source từ repo nào, path trong thư mục đó, Status sync hiện tại

Có thể sync theo thứ tự các component trong application ( deployment,secret,configmap...) dựa vào sync phase : https://argoproj.github.io/argo-cd/user-guide/sync-waves/

**Automated sync**: https://argoproj.github.io/argo-cd/user-guide/auto_sync/

## Repo

Argocd repo server có trách nhiệm quản lý các repo
Ta có thể add thêm các repo vào argocd thông qua web UI => Bản chất khi đó vẫn là argocd repo server sẽ tạo ra secret trong namespace đó với annotation đặc biệt là:
```
  annotations:
    managed-by: argocd.argoproj.io
```

và labels:
```
  labels:
    argocd.argoproj.io/secret-type: repository
```

Nếu ta xóa repository này trên web UI thì bản chất sẽ là xóa secret tương ứng.

Tương tự với các cluster, nếu ta add cluster vào argocd thì bản chất sẽ là secret với label 
argocd.argoproj.io/secret-type: cluster
```
apiVersion: v1
kind: Secret
metadata:
  name: mycluster-secret
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: mycluster.com
  server: https://mycluster.com
  config: |
  ...
```


## Component details
https://argoproj.github.io/argo-cd/operator-manual/architecture/

1. API Server

The API server is a gRPC/REST server which exposes the API consumed by the Web UI, CLI, and CI/CD systems. It has the following responsibilities:

    application management and status reporting
    invoking of application operations (e.g. sync, rollback, user-defined actions)
    repository and cluster credential management (stored as K8s secrets)
    authentication and auth delegation to external identity providers
    RBAC enforcement
    listener/forwarder for Git webhook events

2. Repository Server

The repository server is an internal service which maintains a local cache of the Git repository holding the application manifests. It is responsible for generating and returning the Kubernetes manifests when provided the following inputs:

    repository URL
    revision (commit, tag, branch)
    application path
    template specific settings: parameters, ksonnet environments, helm values.yaml

3. Application Controller

The application controller is a Kubernetes controller which continuously monitors running applications and compares the current, live state against the desired target state (as specified in the repo). It detects OutOfSync application state and optionally takes corrective action. It is responsible for invoking any user-defined hooks for lifecycle events (PreSync, Sync, PostSync)

## Tool Detection

The tool used to build an application is detected as follows:

If a specific tool is explicitly configured, then that tool is selected to create your application's manifests.

If not, then the tool is detected implicitly as follows:

 -   Ksonnet if there are two files, one named app.yaml and one named components/params.libsonnet.
 -   Helm if there's a file matching Chart.yaml.
 -   Kustomize if there's a kustomization.yaml, kustomization.yml, or Kustomization

Otherwise it is assumed to be a plain directory application. 

## Chú ý:
Về việc sử dụng HPA thì ta nên lưu ý, nếu đã sử dụng HPA thì ta add hpa.yml vào repo git đồng thời loại bỏ field replicas trong deployment.yml ngay trước khi chạy sync lần đầu.
quoted from doc:
```
It may be desired to leave room for some imperativeness/automation, and not have everything defined in your Git manifests. For example, if you want the number of your deployment's replicas to be managed by Horizontal Pod Autoscaler, then you would not want to track replicas in Git.
```

ref: https://qiita.com/kikirsch/items/0d6a07697280ffb60e80
https://argoproj.github.io/argo-cd/user-guide/best_practices/
Tham khảo thêm: https://github.com/argoproj/argo-cd/issues/1603

Argo CD polls Git repositories every three minutes to detect changes to the manifests. To eliminate this delay from polling, the API server can be configured to receive webhook events. Argo CD supports Git webhook notifications from GitHub, GitLab, Bitbucket, Bitbucket Server and Gogs. The following explains how to configure a Git webhook for GitHub, but the same process should be applicable to other providers.

https://argoproj.github.io/argo-cd/operator-manual/webhook/

## Sync Waves

Another awesome feature of ArgoCD is Sync Waves. This allows you to synchronously deploy things in a specific order just by attaching an annotation. For instance, you need to deploy Cert Manager before you deploy Rancher, so you can set this annotation within your cert-manager Argo Application:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: kube-system
  annotations:
    argocd.argoproj.io/sync-wave: "-4"

And this annotation for Rancher:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rancher
  namespace: kube-system
  annotations:
    argocd.argoproj.io/sync-wave: "-3"

Since cert-manager's sync wave (-4) is lesser than rancher's (-3), it will deploy first.

