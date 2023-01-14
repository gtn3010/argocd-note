ref:
https://github.com/argoproj/argo-cd/pull/10432

## Adding Multiple Sources
The new field sources would allow users to input list of ApplicationSources. We would ignore the source field we can find even a single entry under sources field.
Below example shows how the yaml would look like for source and sources field. We would ignore the source field and apply the resources mentioned under sources field.

```
spec:
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    path: guestbook
    targetRevision: HEAD
  sources:
    - chart: elasticsearch
      helm:
        valueFiles:
          - values.yaml
      repoURL: https://helm.elastic.co
      targetRevision: 7.6.0
    - repoURL: https://github.com/argoproj/argocd-example-apps.git
      path: guestbook
      targetRevision: HEAD
```

Multiple Value files for Helm Source
For making files accessible to other sources, add a new ref field to the source. For example, in below ApplicationSpec, we have added ref: myRepo to the my-repo repository and have used reference $myRepo to the elasticSearch repository.
```
spec:
  sources:
    - repoURL: https://github.com/my-org/my-repo  # path is missing so no manifests are generated
      targetRevision: master
      ref: myRepo                                 # repo is available via symlink "myRepo"
    - repoURL: https://github.com/helm/charts
      targetRevision: master
      path: incubator/elasticsearch               # path "incubator/elasticsearch" is used to generate manifests
      helm:
        valueFiles:
          - $myRepo/values.yaml                   # values.yaml is located in source with reference name $myRepo
```
