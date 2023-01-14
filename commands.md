curl to trigger webhook argocd sync:
token=<jwt-token>
curl -k -X POST https://<ARGO-URL>/api/v1/applications/<application>/sync --cookie "argocd.token=${token}" --data '{"revision":"master","prune":false,"dryRun":false,"strategy":{},"resources":null}'
