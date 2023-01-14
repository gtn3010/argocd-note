Vì argocd chưa support lấy source từ nhiều repo => tức ta muốn tách 1 helm chart repo và 1 values repo là chưa được.
Ta có thể làm là tạo 1 chart khác , điền subchart là chart platform mà ta muốn deploy VD như mimir/loki...
Ta sẽ điền giá trị ở parent chart => overlay lên subchart

File values sẽ dạng:
```
<subchart_name>:
  test: true
  resources:
  ...
  Các values cần patching overlay từ subchart
```

Vậy nếu ta cũng define 1 resource ở parent chart giống (cùng name và namespace) với resource ở subchart
=> Khi argocd helm template sẽ ra 2 resource object (2 đoạn yaml) cùng name như nhau và cùng namespace

Argocd vẫn sẽ apply tuy nhiên sẽ apply resource generate sau tức resource ở parent chart => Không merge/overlay và trên UI argocd có warning:
```
Resource rbac.authorization.k8s.io/ClusterRole//platform-etcd-test appeared 2 times among application resources.
```

Và đã test ở argocd, trong chart platform mà ta đang để làm subchart lại có 1 subchart nữa (nested subchart)
tức subchart trong subchart thì hiện tại ta sẽ thấy argocd khi helm template sẽ không tự động helm dependency update => pull subchart trong subchart về
