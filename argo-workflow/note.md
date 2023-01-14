Argo workflow controller cũng sẽ tự thực hiện xóa pod (thực thi workflow) sau khi workflow được thực hiện xong.
https://github.com/argoproj/argo-workflows/pull/1234
https://github.com/argoproj/argo-workflows/issues/2173
https://github.com/argoproj/argo-workflows/issues/1230

## Executor

An Argo workflow executor is a process that conforms to a specific interface that allows Argo to perform certain actions like monitoring pod logs, collecting artifacts, managing container lifecycles, etc.

Emissary Executor:
v3.1 introduces a new executor called the emissary. The emissary works by replacing the container's command with its own command. This allows that command to capture stdout, the exit code, and easily terminate your process. The emissary can also delay the start of your process. The container set template allows you to run multiple containers within a workflow pod. This allows the containers to share a workspace.
Starting containers is cheaper and faster than starting pods, so a template that uses a container set can be faster and cheaper.

=> Tương tự như binary ở bên tekton wrap lấy command mà ta define ở từng step => giúp thực hiện các step (container) 1 cách tuần tự VD như đợi khi có 1 file mới thực hiện command ở container tiếp theo.
