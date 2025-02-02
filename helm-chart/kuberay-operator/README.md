# KubeRay Operator

This document provides instructions to install both CRDs (RayCluster, RayJob, RayService) and KubeRay operator with a Helm chart.

## Helm

Make sure the version of Helm is v3+. Currently, [existing CI tests](https://github.com/ray-project/kuberay/blob/master/.github/workflows/helm-lint.yaml) are based on Helm v3.4.1 and v3.9.4.

```sh
helm version
```

## Install CRDs and KubeRay operator

* Install a stable version via Helm repository (only supports KubeRay v0.4.0+)
  ```sh
  helm repo add kuberay https://ray-project.github.io/kuberay-helm/

  # Install both CRDs and KubeRay operator v0.4.0.
  helm install kuberay-operator kuberay/kuberay-operator --version 0.4.0

  # Check the KubeRay operator Pod in `default` namespace
  kubectl get pods
  # NAME                                READY   STATUS    RESTARTS   AGE
  # kuberay-operator-6fcbb94f64-mbfnr   1/1     Running   0          17s
  ```

* Install the nightly version
  ```sh
  # Step1: Clone KubeRay repository

  # Step2: Move to `helm-chart/kuberay-operator`

  # Step3: Install KubeRay operator
  helm install kuberay-operator .
  ```

## List the chart

To list the `my-release` deployment:

```sh
helm ls
# NAME                    NAMESPACE       REVISION        UPDATED                                   STATUS          CHART                   # APP VERSION
# kuberay-operator        default         1               2022-12-02 02:13:37.514445313 +0000 UTC   deployed        kuberay-operator-0.4.0  1.0
```

## Uninstall the Chart

```sh
# Uninstall the `kuberay-operator` release
helm uninstall kuberay-operator

# The operator Pod should be removed.
kubectl get pods
# No resources found in default namespace.
```

## Working with Argo CD

If you are using [Argo CD](https://argoproj.github.io) to manage the operator, you will encounter the issue which complains the CRDs too long. Same with [this issue](https://github.com/prometheus-operator/prometheus-operator/issues/4439).
The recommended solution is to split the operator into two Argo apps, such as:

* The first app just for installing the CRDs with `Replace=true` directly, snippet:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ray-operator-crds
spec:
  project: default
  source:
    repoURL: https://github.com/ray-project/kuberay
    targetRevision: v0.4.0
    path: helm-chart/kuberay-operator/crds
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    syncOptions:
    - Replace=true
...
```

* The second app that installs the Helm chart with `skipCrds=true` (new feature in Argo CD 2.3.0), snippet:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ray-operator
spec:
  source:
    repoURL: https://github.com/ray-project/kuberay
    targetRevision: v0.4.0
    path: helm-chart/kuberay-operator
    helm:
      skipCrds: true
  destination:
    server: https://kubernetes.default.svc
    namespace: ray-operator
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
...
```
