https://argo-cd.readthedocs.io/en/stable/operator-manual/security/
https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/

## Tóm tắt

Argocd sử dụng dex phục vụ cho SSO
Ta kiểm tra pod dex server sẽ thấy có 2 process:
```
    1 999       0:00 /shared/argocd-dex rundex
   27 999       0:00 dex serve /tmp/dex.yaml
```

Binary /shared/argocd-dex sẽ đảm nhiêm việc watch vào configmap argocd-cm để liên tục cập nhật file config của dex ở /tmp/dex.yaml.
Và khi config mới được cập nhật ( file /tmp/dex.yaml có sự thay đổi ) thì binary/process này cũng tiến hành load lại sub process dex.
Check log ta sẽ thấy process dex bị shutdown và start lại.

Chú ý là khi đó pod UI argocd server bị mất kết nối đến dex => trên UI web sẽ liên tục bắn ra lỗi.
Ta có thể restart pod argocd server này.


## Authentication
Authentication to Argo CD API server is performed exclusively using JSON Web Tokens (JWTs). Username/password bearer tokens are not used for authentication. The JWT is obtained/managed in one of the following ways:

For the local admin user, a username/password is exchanged for a JWT using the /api/v1/session endpoint. This token is signed & issued by the Argo CD API server itself, and has no expiration. When the admin password is updated, all existing admin JWT tokens are immediately revoked. The password is stored as a bcrypt hash in the argocd-secret Secret.

For Single Sign-On users, the user completes an OAuth2 login flow to the configured OIDC identity provider (either delegated through the bundled Dex provider, or directly to a self-managed OIDC provider). This JWT is signed & issued by the IDP, and expiration and revocation is handled by the provider. Dex tokens expire after 24 hours.

Automation tokens are generated for a project using the /api/v1/projects/{project}/roles/{role}/token endpoint, and are signed & issued by Argo CD. These tokens are limited in scope and privilege, and can only be used to manage application resources in the project which it belongs to. Project JWTs have a configurable expiration and can be immediately revoked by deleting the JWT reference ID from the project role.

## Authorization
Authorization is performed by iterating the list of group membership in a user's JWT groups claims, and comparing each group against the roles/rules in the RBAC policy. Any matched rule permits access to the API request.

## Notice

scopes controls which OIDC scopes to examine during rbac enforcement (in addition to `sub` scope).
If omitted, defaults to: '[groups]'. The scope value can be a string, or a list of strings.
scopes: '[cognito:groups, email]'

Scope when we set configmap of argo server:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
spec:
  data:
    policy.csv: |
      g, nguyengt@g-pay.vn, role:org-admin
    scopes: "[email,groups]"
    policy.default: role:readonly
```

Argocd use field get from JWT of user to check rbac. We can set which fields argocd will iterate for checking. Like email or groups...

