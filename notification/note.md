Arogcd notification controler sẽ watch vào configmap có name: argocd-notifications-cm
để cập nhật động config về template notification
=> Check trong role ta sẽ thấy service account của pod argocd notifications được bind vào role có quyền trên resource configmap với name là argocd-notifications-cm

Ta có thể test generate notification template và in ra console:
docker run --rm -it --name argonoti -v /tmp/kubeconfigtest/:/kubeconfig argoprojlabs/argocd-notifications:v1.2.1 /app/argocd-notifications --kubeconfig /kubeconfig/config template notify app-sync-succeeded platform-gitlab-runner-gitops-commit

=> Command này sẽ kết nối lên pod argocd-notification trên k8s cluster bằng file kubeconfig mà ta cung cấp

read more:
https://argocd-notifications.readthedocs.io/en/stable/troubleshooting/