repo-server là component sẽ thực hiện pull từ git về và generate manifest

cụ thể là với helm thì sẽ thực hiện chạy helm dependency build và helm template ...
Sau đó pipe output yaml đó vào kubectl apply.

Nếu ở trong helm chart của applications có sub chart/dependency chart và sub chart này chưa được pull về cùng thư mục với chart chính (chưa có thư mục charts, chưa có file chart.lock) thì argocd repo server trước tiên sẽ thực hiện add repo (để check helm chart repo có exist) và pull về /tmp với `helm repo add --insecure-skip-tls-verify application-helm-chart git@gitlab.g-pay.vn:devops/helm/applications.git`
trong đó application-helm-chart là field name khi ta tạo repository ở argocd
=> Vì vậy cả helm chart chính và dependency chart đều cần được đưa vào helm repo (chứ không phải git repo, bởi khi đó command trên sẽ failed).
Sau đó mới thực hiện pull dependency chart bằng command "helm dependency build" và sau đó mới thực hiện helm template

Đôi lúc ta thấy trong /tmp có thư mục dạng như này /tmp/helm2440349 => Đây là thư mục tạm (temp dir) khi argocd repo server thực hiện repoAdd (helm repo add)
ref:
https://github.com/argoproj/argo-cd/blob/32838b1f71718f7f7f4e134d68b25c34014085ea/util/helm/cmd.go#L144

Với source code của repo server:
https://github.com/argoproj/argo-cd/blob/32838b1f71718f7f7f4e134d68b25c34014085ea/reposerver/repository/repository.go

Ta cần chú ý 4 file sau:
	helmDepUpMarkerFile            = ".argocd-helm-dep-up"
	allowConcurrencyFile           = ".argocd-allow-concurrency"
	repoSourceFile                 = ".argocd-source.yaml"
	appSourceFile                  = ".argocd-source-%s.yaml"

Trong đó:
.argocd-helm-dep-up: đây là helm dependency markup file => Tức file này sẽ để đánh dấu cho argocd repo server biết là đã thực hiện chạy command `helm dependecy build` rồi và không cần thực hiện chạy lại nữa. File này thường do argocd server tự tạo.
```
// runHelmBuild executes `helm dependency build` in a given path and ensures that it is executed only once
// if multiple threads are trying to run it.
// Multiple goroutines might process same helm app in one repo concurrently when repo server process multiple
// manifest generation requests of the same commit.
func runHelmBuild(appPath string, h helm.Helm) error {
	manifestGenerateLock.Lock(appPath)
	defer manifestGenerateLock.Unlock(appPath)

	// the `helm dependency build` is potentially time consuming 1~2 seconds
	// marker file is used to check if command already run to avoid running it again unnecessary
	// file is removed when repository re-initialized (e.g. when another commit is processed)
	markerFile := path.Join(appPath, helmDepUpMarkerFile)
...
```
Tuy nhiên file này sẽ chỉ được tạo khi ta allow concurrency => nhiều app chung chart directory
```
		if concurrencyAllowed {
			err = runHelmBuild(appPath, h)
		} else {
			err = h.DependencyBuild()
		}
...
func runHelmBuild(appPath string, h helm.Helm) error {
	manifestGenerateLock.Lock(appPath)
	defer manifestGenerateLock.Unlock(appPath)

	// the `helm dependency build` is potentially time consuming 1~2 seconds
	// marker file is used to check if command already run to avoid running it again unnecessary
	// file is removed when repository re-initialized (e.g. when another commit is processed)
	markerFile := path.Join(appPath, helmDepUpMarkerFile)
	_, err := os.Stat(markerFile)
	if err == nil {
		return nil
	} else if !os.IsNotExist(err) {
		return err
	}

	err = h.DependencyBuild()
	if err != nil {
		return err
	}
	return ioutil.WriteFile(markerFile, []byte("marker"), 0644)
}
```
ref:
https://github.com/argoproj/argo-cd/blob/32838b1f71718f7f7f4e134d68b25c34014085ea/reposerver/repository/repository.go#L534


.argocd-allow-concurrency
=> File này ta tạo trong chart directory để enable concurrent processing cho 2 application sử dụng chung 1 chart. Nên cân nhắc sử dụng tính năng này. Lí do sẽ được nói ở dưới
https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/#enable-concurrent-processing
Argo CD determines if manifest generation might change local files in the local repository clone based on config management tool and application settings. If the manifest generation has no side effects then requests are processed in parallel without the performance penalty. Following are known cases that might cause slowness and workarounds:
Multiple Helm based applications pointing to the same directory in one Git repository: ensure that your Helm chart don't have conditional dependencies and create .argocd-allow-concurrency file in chart directory.

.argocd-source.yaml và .argocd-source-%s.yaml
https://argo-cd.readthedocs.io/en/stable/user-guide/parameters/
Argo CD provides a mechanism to override the parameters of Argo CD applications that leverages config management tools. This provides flexibility in having most of the application manifests defined in Git, while leaving room for some parts of the k8s manifests determined dynamically, or outside of Git. It also serves as an alternative way of redeploying an application by changing application parameters via Argo CD, instead of making the changes to the manifests in Git.

=> Ta có thể override các params của application bằng cách điền các values tương ứng vào file này và đặt trong chart directory.

Chú ý
Các repo khi pull về sẽ được lưu ở /tmp (VD: /tmp/git@gitlab.g-pay.vn_devops_helm_platform). Mỗi repo sẽ là 1 thư mục con trong /tmp. Dù cho các app có cùng repo nhưng khác revision (branch,commit) khác nhau thì khi pull về vẫn được đặt chung trong cùng 1 thư mục của repo tương ứng này trong /tmp. 2 hoặc nhiều app có thể dùng chung repo, có thể cùng hoặc khác revision (branch/commit) và có thể chung hoặc khác chart directory nằm trong repo này. Tuy nhiên nếu 2 app cùng chung repo và revision và cùng chart directory, mà ta sử dụng configmanagementplugin , có script sửa file chung trong chart directory tương ứng này mà ta tạo file .argocd-allow-concurrency để enable concurrency processing thì cần cẩn thận đến race condition, 2 app cùng sửa file values dẫn đến có thể app nọ sửa xong, bị app khác override mất và sử dụng values của app khác. (Vì vậy với trường hợp app platform-ambassador-public và platform-ambassador-private trước đây ta tạo script để chèn thêm static IP public externalvào service object , ta đã sửa cùng vào file values-overlay.yaml trong chart directory => Dẫn đến nếu app platform-ambassador-public chạy sau, script sẽ sửa thành IP public của platform-ambassador-public và từ đó platform-ambassador-private sync lại sẽ sử dụng values của platform-ambassador-public => Sai => K ổn định)
Nên hạn chế trường hợp sử dụng 2 app cùng chung repo, chung revision, chung **CHART DIRECTORY** và có plugin updates vào cùng 1 file values.

Tuy nhiên thường manifest sau khi được generate ra sẽ được cache lại trong redis , trường hợp trên dễ xảy ra khi ta chạy force sync hoặc concurrency nhiều app => race condition
Vậy nên dù có cùng repo, mà khác revision , ở các revision khác nhau lại update,xóa,thêm file khác nhau mà chỉ lưu chung vào 1 thư mục /tmp/<repo-name...> lại không gây ảnh hưởng giữa các app (Do đã cache manifest generate ra rồi). Và manifest này được cache cũng có ttl, hết ttl argocd repo server có thể sẽ thực hiện pull lại hoặc nếu ta lên giao diện force sync lại toàn bộ application thì quá trình pull này cũng được tiến hành lại => Khi đó vào /tmp/<repo-name> ta cũng sẽ thấy thư mục này nội dung khác đi.


1 điểm nữa là như ta đã thấy nếu app đó sử dụng repo với helm chart thì argocd repo server cũng đã có step chạy `helm dependency build` trước khi helm template giúp mình. => Vì vậy có thể không cần tự viết plugin để pull dependency lib chart về nữa.


## From doc
ref:
https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/#argocd-repo-server

The argocd-repo-server is responsible for cloning Git repository, keeping it up to date and generating manifests using the appropriate tool.

argocd-repo-server fork/exec config management tool to generate manifests. The fork can fail due to lack of memory and limit on the number of OS threads. The --parallelismlimit flag controls how many manifests generations are running concurrently and allows avoiding OOM kills.

**the argocd-repo-server ensures that repository is in the clean state during the manifest generation using config management tools such as Kustomize, Helm or custom plugin. As a result Git repositories with multiple applications might be affect repository server performance. Read Monorepo Scaling Considerations for more information.**

argocd-repo-server clones repository into /tmp ( of path specified in TMPDIR env variable ). Pod might run out of disk space if have too many repository or repositories has a lot of files. To avoid this problem mount persistent volume.

argocd-repo-server git ls-remote to resolve ambiguous revision such as HEAD, branch or tag name. This operation is happening pretty frequently and might fail. To avoid failed syncs use ARGOCD_GIT_ATTEMPTS_COUNT environment variable to retry failed requests.

argocd-repo-server Every 3m (by default) Argo CD checks for changes to the app manifests. Argo CD assumes by default that manifests only change when the repo changes, so it caches generated manifests (for 24h by default). With Kustomize remote bases, or Helm patch releases, the manifests can change even though the repo has not changed. By reducing the cache time, you can get the changes without waiting for 24h. Use --repo-cache-expiration duration, and we'd suggest in low volume environments you try '1h'. Bear in mind this will negate the benefit of caching if set too low.

argocd-repo-server fork exec config management tools such as helm or kustomize and enforces 90 seconds timeout. The timeout can be increased using ARGOCD_EXEC_TIMEOUT env variable.