ref:
https://github.com/argoproj/argo-cd/issues/9332
https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/high_availability.md

Since the V1.4.0 version, the kubectl fork/exec method is no longer used but replaced by the client-go in controller, so documentation should be updated.
https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/server-commands/argocd-application-controller.md

Old version:
The controller uses kubectl fork/exec to push changes into the cluster and to convert resource from preferred version into user specified version (e.g. Deployment apps/v1 into extensions/v1beta1). Same as config management tool kubectl fork/exec might cause pod OOM kill. Use --kubectl-parallelism-limit flag to limit number of allowed concurrent kubectl fork/execs.
