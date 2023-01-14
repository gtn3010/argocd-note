Ở hệ thống hiện tại của gpay, đã thử xóa toàn bộ application đi (xóa root application => xóa applicationsets => xóa applications)
Về cơ bản PVC và PV vẫn còn, do data của vault không bị ảnh hưởng.
Gặp vấn đề bị treo khi xóa các resource, đặc biệt là resource của ambassador (do webhook mutating pod và service đã bị xóa mất) mà argocd cần list các api resource trên k8s ra nhưng do pod webhook mutating của ambassador (api ext crd: Phục vụ cho việc migrate version CRD v2 và v3) bị xóa mất, dẫn đến không thể get ra được các api resource (Check status của application object sẽ thấy).
