The responsibility of the ApplicationSet controller is to convert an ApplicationSet resource into one or more Application resources. However, with the previous releases, if at least one of the generated Application resources was invalid (e.g. it failed the internal validation logic), none of the generated Applications would be processed (they would not be neither created nor modified).

With the latest ApplicationSet release, if a generator generates invalid Applications, those invalid generated Applications will still be skipped, but the valid generated Applications will now be processed (created/modified).

Thus no ApplicationSet resource changes are required by this new behaviour, but it is worth keeping in mind that your ApplicationSets which were previously blocked by a failing Application may no longer be blocked. This change might cause valid Applications to now be created/modified, whereas previously they were prevented from being processed.

ref:
https://github.com/argoproj-labs/applicationset/releases/tag/v0.3.0

## Post Selector all generators¶
The Selector allows to post-filter based on generated values using the kubernetes common labelSelector format. In the example, the list generator generates a set of two application which then filter by the key value to only select the env with value staging:

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - list:
      elements:
        - cluster: engineering-dev
          url: https://kubernetes.default.svc
          env: staging
        - cluster: engineering-prod
          url: https://kubernetes.default.svc
          env: prod
    selector:
      matchLabels:
        env: staging
  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj-labs/applicationset.git
        targetRevision: HEAD
        path: examples/list-generator/guestbook/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: guestbook
```

ref:
https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/#post-selector-all-generators

## Go template

https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/GoTemplate/

ApplicationSet is able to use Go Text Template. To activate this feature, add goTemplate: true to your ApplicationSet manifest.
