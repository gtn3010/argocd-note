## Like fluxCD , Support `[skip ci]` / `[ci skip]` in commit messages 

ref:
https://github.com/argoproj/argo-cd/issues/878

This use-case is solved using --targetRevision parameter. The targetRevision defines commit, tag, or branch in which to sync the application to. E.g. if application's targetRevision is set to production then developers might update master any time and release changes to production once master is stable.

Argo CD reconciles app state in response to multiple events: any change in target cluster, commit in repo, periodically after some time. So there is just no place in code where Argo CD could check for commit message.

Your point about overhead is valid: argo cd should not regenerate manifests/hit k8s api unnecessary. We are caching generated manifests as well as target cluster state to avoid making unnecessary API calls.


## Change cluster in application object

ref:
https://www.reddit.com/r/kubernetes/comments/nbkkn2/argocd_chart_purging_question/

Q:
I'm using argocd to deploy to multiple kubernetes clusters. I noticed that when i deploy a helm chart with destination name: dev the chart provisions just fine on the dev cluster, then switching the destination name: prod provisions the chart on the prod cluster just fine, but argo does not clean up the old chart on dev. is this expected behavior?

A:
- If you deploy to a cluster named "dev", then there is no problem: it works.
- If you change anything, ArgoCD will update the state accordingly. No problem.
- If you change the cluster name though, it has 2 choices:
    - rename the cluster, which is likely not supported as it's rarely what you want, or
    - use a cluster named "prod", which has no deployment yet, so ArgoCD deploys there. The cluster named "dev" is not touched though. ArgoCD does not care about it.

So you would have to manually delete it on the cluster named "dev". Which honestly seems to be most reasonable.
