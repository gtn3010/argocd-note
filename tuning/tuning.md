Namespace of resource in Application:
Argo CD honors namespace specified in the resource to unblock deployment of complex applications, such as the prometheus-operator, where all resources are supposed to be deployed into application namespace and few resources are deployed into hard-coded namespace.

=> If we specify namespace in resource k8s in git, argocd will deploy/sync it to this namespace (not in namespace we specify in destination field of Application object)
ref:
https://github.com/argoproj/argo-cd/issues/2280


Parameters override:
https://argo-cd.readthedocs.io/en/stable/user-guide/parameters/


Use with argo rollout:
https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/#known-kubernetes-types-in-crds-resource-limits-volume-mounts-etc

Sync only outofsync resource:
https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#selective-sync

Prune resource happen final:
https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#prune-last


Webhook and Manifest Paths Annotation
https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/#webhook-and-manifest-paths-annotation
Argo CD aggressively caches generated manifests and uses repository commit SHA as a cache key. A new commit to the Git repository invalidates cache for all applications configured in the repository that again negatively affect mono repositories with multiple applications. You might use webhooks ⧉ and argocd.argoproj.io/manifest-generate-paths Application CRD annotation to solve this problem and improve performance.

The argocd.argoproj.io/manifest-generate-paths contains a semicolon-separated list of paths within the Git repository that are used during manifest generation. The webhook compares paths specified in the annotation with the changed files specified in the webhook payload. If non of the changed files are located in the paths then webhook don't trigger application reconciliation and re-uses previously generated manifests cache for a new commit.



Application controller resync default every 180s (3mins), app state cache default (1hour)


Repo server:
Every 3m (by default) Argo CD checks for changes to the app manifests.
parallelismlimit: Limit on number of concurrent manifests generate requests. Any value less the 1 means no limit.
repo-cache-expiration duration : Cache expiration for repo state, incl. app lists, app details, manifest generation, revision meta-data (default 24h0m0s)
revision-cache-expiration duration : Cache expiration for cached revision (default 3m0s)

ref more:
https://medium.com/@keska.damian/optimizing-argocd-repo-server-to-work-with-kustomize-in-monorepo-974c3a97334d


Replacing --app-resync flag with timeout.reconciliation setting
The--app-resync flag allows controlling how frequently Argo CD application controller checks resolve the target application revision of each application. In order to allow caching resolved revision per repository as opposed to per application, the --app-resync flag has been deprecated. Please use timeout.reconciliation setting in argocd-cm ConfigMap instead. The value of timeout.reconciliation is a duration string e.g 60s, 1m, 1h or 1d. See example in argocd-cm.yaml.
=> timeout.reconciliation config in configmap argocd-cm.yaml

```
  # Application reconciliation timeout is the max amount of time required to discover if a new manifests version got
  # published to the repository. Reconciliation by timeout is disabled if timeout is set to 0. Three minutes by default.
  # > Note: argocd-repo-server deployment must be manually restarted after changing the setting.
  timeout.reconciliation: 180s
```
more:
https://github.com/argoproj/argo-helm/issues/1174
