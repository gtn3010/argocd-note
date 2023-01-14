https://medium.com/@hpanay/gitops-and-security-3ce436fe88c3
https://argoproj.github.io/argo-cd/user-guide/config-management-plugins/
https://argoproj.github.io/argo-cd/operator-manual/custom_tools/


## Integration into GitOps

Now that we’ve picked our tool, we can integrate it into GitOps. Our controller of choice is ArgoCD so we know that will need the rights to decrypt manifests as well as the Sops binary to do that with. Here’s how we add the binary to the repo server component:

```
initContainers:
      - name: download-sops
        image: alpine:3.8
        command: [sh, -c]
        args:
        - wget https://github.com/mozilla/sops/releases/download/v3.5.0/sops-v3.5.0.linux && mv sops-v3.5.0.linux /custom-tools/sops && chmod +x /custom-tools/sops
        volumeMounts:
        - name: custom-tools
          mountPath: /custom-tools
```

Sops uses the gcloud sdk, therefore we need to mount the creds and specify their location:

```
...
env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /app/config/gcp/gcp-creds.json
...
volumes:      
        - secret:
          secretName: gcp-creds
          name: gcp-creds
```

Of course, the IAM service account will need the cloudkms.cryptoKeyDecrypter GCP role (dammit medium, fix how you do inline codeblocks please).

Now that we’ve given ArgoCD the rights, we need to modify the process so that it’s taken into account. ArgoCD simply looks for a kube manifest within the directory we told it to look at but now it’s encrypted. Thankfully, ArgoCD allows you to add custom plugins via the configmap:
```
configManagementPlugins: |
    - name: sops
      generate:
        command: [sh, -c]
        args: ["sops -d --input-type=json --output-type=json $MANIFEST_PATH"]
```

The job of a custom plugin is to output a Kubernetes valid manifest via stdout. Once it does that, ArgoCD will apply it will take output and process it. That sops command will do just that. But where does $MANIFEST_PATH come from?

ArgoCD is driven by ArgoCD custom Kubernetes resources. This custom resource has a section for defining where things come from. The section is called source and looks like this:
```
...
  "source": {
      "repoURL": <gitrepo>,
      "targetRevision": <revision_of_git_repo>,
      "path": <directory_of_manifests>,
      "plugin": {
        "name": "sops",
        "env": [
          {
            "name": "MANIFEST_PATH",
            "value": "manifest.json"
          }
        ]
      }
    },
...
```

As you can see between this and the command, the working directory is the directory specified so we don’t need to build the path up. The environment key can contain any manner of key value pairs, we’re using that for defining the manifest


## More example

https://github.com/argoproj/argo-cd/issues/1364