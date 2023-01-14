Argo Rollouts Controller needs to know which Service is the canaryService and which is the stableService so it can update them to select their corresponding pods on the fly;
In order for us to know which pods are being selected by which service, we can specify a label or annotation to be attached to the Pods metadata via canaryMetadata and stableMetadata;
Argo Rollouts Controller also needs to know which Ingress is the stableIngress so it can create automatically the canary Ingress from it;
Finally, in order to avoid us from manually editing the Ingress’s canary-weight annotation, the Argo Rollouts Controller will be able to do it for us automatically, thanks to the steps specification.

```
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
...
spec:
  ...
  strategy:
    canary:
      canaryService: {{ .Release.Name }}-canary
      stableService: {{ .Release.Name }}
      canaryMetadata:
        labels:
          deployment: canary
      stableMetadata:
        labels:
          deployment: stable
      trafficRouting:
        nginx:
          stableIngress: {{ .Release.Name }}
          additionalIngressAnnotations:
            canary-by-header: X-Canary
      steps:
        - setWeight: 25
        - pause: {}
        - setWeight: 50
        - pause:
            duration: 5m
        - setWeight: 75
        - pause:
            duration: 5m
  ...

```


