Argo CD has several levels of caching to minimize the number of interactions with Git:
- The git repo content is cached on the argocd-repo-server file system, so that Argo CD only needs to do a git fetch when a new commit is pushed to the repo.
- Generated application manifests are stored in a shared Redis cache, so that the Kubernetes config management tool is executed only once per commit.
- However, even with the above caching mechanisms, Argo CD still needs to resolve the HEAD, branch name or tag to the current commit SHA for each and every application reconciliation, which results in thousands of Git queries per day.

In v1.1 the reconciliation logic was improved so that the commit SHA is resolved only in one of the following cases:
- Once per every application re-sync period (3 minutes by default)
- When the user explicitly requests application refresh ( e.g. using argocd app get <appName> — refresh` )
- When Argo CD is notified of a new commit via a WebHook

These optimizations significantly reduced the number of Git queries. At Intuit, we are seeing a reduction of ~4–5x in the number of Git queries.

ref:
https://blog.argoproj.io/argo-cd-v1-1-released-8290aed6ad66
